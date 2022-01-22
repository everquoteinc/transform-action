# transform-action
GitHub Action to transform the contents of a directory

## Usage

```yaml
permissions:
  checks: write
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Check out repository
      uses: actions/checkout@v2
    - name: Install required software for .render scripts
      run: |
        sudo snap install kustomize
        sudo snap install helm --classic
    - name: Generate deployment manifests
      id: transform
      uses: everquoteinc/transform-action@v1
      with:
        source-dir: config
        target-dir: deploy
        copy-file-pattern: '*.yaml'
        rendered-file-name: manifest.yaml
    - name: Push changes to branch
      uses: everquoteinc/git-push-action@v1
      with:
        commit-message: add auto-generated deployment manifests
        file-pattern: 'deploy/'
      if: steps.transform.outcome == 'success'
    - name: Set status
      shell: bash
      env:
        GITHUB_REPO: ${{ github.repository }}
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      run: |
        BUILD_COMMIT=$(git rev-parse HEAD)
        DATA=$(printf '{"name":"deploy","head_sha":"%s","status":"completed","conclusion":"success"}' "$BUILD_COMMIT")
        curl --silent \
             --header "Authorization: Bearer $GITHUB_TOKEN" \
             --header "Accept: application/vnd.github.v3+json" \
             --header "Content-Type: application/json" \
             --request "POST" --data "$DATA" \
             "https://api.github.com/repos/$GITHUB_REPO/check-runs"
      if: steps.push-changes.outcome == 'success'
```

## Inputs

This action accepts the following input parameters:

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `source-dir` | true | | Source directory |
| `target-dir` | true | | Target directory |
| `max-depth`  | false | `10` | Maximum recursion depth |
| `copy-file-pattern` | false | `*` | Copy file pattern |
| `rendered-file-name` | false | `rendered` | Render output file name |

## Outputs

None.

## Logic

This action recurses through `<source-dir>` up to a max depth of `<max-depth>`.

The directory structure of `<source-dir>` is replicated to `<target-dir>` except
for subdirectories that are pruned by special files.

The following files in the source directory govern the behavior of this action
(in descending order of priority):

| File Name | Behavior |
|-----------|----------|
| `.ignore` | Ignore all files in directory. Do not descend into subdirectories. |
| `.render` | Execute `.render` file (which must be executable) and send STDOUT to `<rendered-file-name>` in `<target-dir>`. Ignore all other files in directory. Do not descend into subdirectories. |
| `<copy-file-pattern>` | Copy files to `<target-dir>`. |

## Example

| Parameter | Value |
|-----------|-------|
| `source-dir` | `config` |
| `target-dir` | `deploy` |
| `max-depth`  | `2` |
| `copy-file-pattern` | `*.yaml` |
| `rendered-file-name` | `manifest.yaml` |

The above input parameters would result in the following transformation:

```
config
|__ node1
|   |__ .ignore
|   |__ .render
|   |__ app.yaml
|   |__ component
|       |__ a.yaml
|       |__ b.yaml
|__ node2
|   |__ .render
|   |__ app.yaml
|   |__ component
|       |__ a.yaml
|       |__ b.yaml
|__ node3
    |__ app.md
    |__ app.yaml
    |__ component
        |__ a.yaml
        |__ b.yaml
        |__ foo
            |__ c.yaml
```

->

```
deploy
|__ node2
|   |__ manifest.yaml
|__ node3
    |__ app.yaml
    |__ component
        |__ a.yaml
        |__ b.yaml
```

The `.render` file in the above example could be something like this:

```bash
#!/usr/bin/env bash

kustomize build .
```

