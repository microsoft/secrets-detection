# Gitleaks

## Overview

In this article, you can find information on integrating [Gitleaks](https://github.com/zricethezav/gitleaks) into your GitHub Workflow or Azure Pipeline as part of the PR and Build process.

## What is Gitleaks?

Gitleaks is an Open-Source tool released under [MIT License](https://opensource.org/licenses/MIT). It's a [SAST](../../../03-Deploy/Static-Application-Security-Testing/) tool for detecting and preventing hardcoded secrets like passwords, API keys, and tokens in git repos. Gitleaks is an easy-to-use, all-in-one solution for detecting secrets, past or present, in your code.

Gitleaks produces a scan report as [SARIF (Static Analysis Results Interchange Format)](http://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html) file, which is [OASIS](https://www.oasis-open.org/) standard used to streamline how static analysis tools share their results.

## GitHub Workflows

In this section, you can find example recipes for GitHub Workflow.

### Option 1 - Gitleaks with official GitHub Action

Gitleaks officially released its GitHub Action called Gitleaks [Gitleaks Action](https://github.com/marketplace/actions/gitleaks) to perform secret detection, and it's recommended and the easiest solution to use Gitleaks in the GitHub Workflow.

> **NOTE**
>
> Gitleaks Action is released under a commercial license!

- If you are scanning repos that belong to an **organization account**, you will need to obtain a license key.
- If you are scanning repos belonging to a **personal account**, no license key is required.

For more information about licensing, go to the official page: [gitleaks.io](https://gitleaks.io/products.html)

Gitleaks result (SARIF file) may be uploaded to the [GitHub Code Scanning](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/about-code-scanning) service to see code scanning alerts from third-party tools.

> **NOTE**
>
> **GitHub Code Scanning** integration it's only available for Organizations that use [GitHub Enterprise Cloud](https://docs.github.com/en/get-started/learning-about-github/githubs-products#github-enterprise) and have a license for [GitHub Advanced Security](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security).

{% raw %}

```yaml
name: Gitleaks-GHA-official

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]
  workflow_dispatch:

jobs:
  secrets-detection:
    name: Secrets Detection
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Run Gitleaks
        id: gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }} # Only required for Organizations, not personal accounts.

      # Only for Organizations that use GitHub Enterprise Cloud and have a license for GitHub Advanced Security.
      - name: Upload SARIF report to Code Scanning service
        if: always() && steps.gitleaks.outputs.exit-code != 0
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: results.sarif
          category: gitleaks
```

{% endraw %}

To adjust **Gitleaks Action** configuration, follow documentation: <https://github.com/marketplace/actions/gitleaks#environment-variables>

### Option 2 - Gitleaks with unofficial GitHub Action

There are a couple of unofficial Gitleaks GitHub Actions on the [GitHub Marketplace](https://github.com/marketplace?query=gitleaks). In most cases, they do not require a commercial license and are released under open-source licenses.

Each Action has a different approach for Gitleaks usage and scanning, so there is not one answer to what is the best - it depends on scenario requirements.

> **NOTE**
>
> Very often, unofficial Actions are community-powered and may have undefined code maintenance and support approaches.

One community-based GitHub Action has been selected for this example, which has similar results and experience as the official one - [Gitleaks Scanner Action](https://github.com/marketplace/actions/gitleaks-scanner).

{% raw %}

```yaml
name: Gitleaks-GHA-unofficial

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]
  workflow_dispatch:

jobs:
  secrets-detection:
    name: Secrets Detection
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Run Gitleaks
        id: gitleaks
        uses: DariuszPorowski/github-action-gitleaks@v2
        with:
          report_format: sarif

      - name: Upload SARIF report to Workflow Artifacts
        uses: actions/upload-artifact@v3
        if: always() && steps.gitleaks.outputs.exitcode != 0
        with:
          name: gitleaks
          path: |
            ${{ steps.gitleaks.outputs.report }}

      # Only for Organizations that use GitHub Enterprise Cloud and have a license for GitHub Advanced Security.
      - name: Upload SARIF report to Code Scanning service
        if: always() && steps.gitleaks.outputs.exitcode != 0
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: ${{ steps.gitleaks.outputs.report }}
          category: gitleaks
```

{% endraw %}

To adjust **Gitleaks Scanner Action** configuration, follow documentation: <https://github.com/marketplace/actions/gitleaks-scanner#inputs>

### Option 3 - Gitleaks as command line

You can run Gitleaks natively as a command line without using any GitHub Actions. In this scenario, you have complete control, but you are responsible for code maintenance based on Gitleaks breaking changes in future releases. Moreover, you have to setup Gitleaks on the agent as well.

The following example contains the setup step that downloads and installs the latest version of the Gitleaks from the official repository and the step with Gitleaks execution to detect secrets in the code.

{% raw %}

```yaml
name: Gitleaks-CMD

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]
  workflow_dispatch:

jobs:
  secrets-detection:
    name: Secrets Detection
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Setup Gitleaks
        run: |
          curl -sSLO $(curl -sSL https://api.github.com/repos/zricethezav/gitleaks/releases/latest | jq --compact-output --raw-output '.assets[] | select( .name | contains("linux_x64") ) | select( .name | endswith(".tar.gz") ).browser_download_url')
          filename=$(find . -maxdepth 1 -type f -regex '^.*\/*linux_x64*\.tar.gz')
          tar -xf "${filename}" gitleaks
          mv -f gitleaks /usr/local/bin/
          rm -f "${filename}"

      - name: Run Gitleaks
        id: gitleaks
        run: |
          set +e
          report_name="gitleaks-report"
          report_format="sarif"

          command="gitleaks detect --redact --verbose --report-format=${report_format} --report-path=${report_name}.${report_format} --log-level debug"
          echo "Running Gitleaks $(gitleaks version)"
          echo "${command}"
          OUTPUT=$(eval "${command}")
          exitcode=$?

          echo "::set-output name=command::${command}"
          echo "::set-output name=exitcode::${exitcode}"

          if [ ${exitcode} -eq 0 ]; then
            GITLEAKS_RESULT="SUCCESS! Your code is good to go"
            echo "::notice::${GITLEAKS_RESULT}"
          elif [ ${exitcode} -eq 1 ]; then
            GITLEAKS_RESULT="STOP! Gitleaks encountered leaks or error"
            echo "::error::${GITLEAKS_RESULT}"
          else
            echo "::error::Gitleaks unknown error"
            exit ${exitcode}
          fi

          echo "${OUTPUT}"

          echo "Gitleaks Summary: ${GITLEAKS_RESULT}" >>$GITHUB_STEP_SUMMARY
          echo "${OUTPUT}" >>$GITHUB_STEP_SUMMARY

          echo "::set-output name=report::${report_name}.${report_format}"

          exit ${exitcode}

      - name: Upload SARIF report to Workflow Artifacts
        uses: actions/upload-artifact@v3
        if: always() && steps.gitleaks.outputs.exitcode != 0
        with:
          name: gitleaks
          path: |
            ${{ steps.gitleaks.outputs.report }}

      # Only for Organizations that use GitHub Enterprise Cloud and have a license for GitHub Advanced Security.
      - name: Upload SARIF report to Code Scanning service
        if: always() && steps.gitleaks.outputs.exitcode != 0
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: ${{ steps.gitleaks.outputs.report }}
          category: gitleaks
```

{% endraw %}

To adjust **Gitleaks** configuration, follow documentation: <https://github.com/zricethezav/gitleaks#usage>

### GitHub References

- [Gitleaks Action](https://github.com/marketplace/actions/gitleaks) (official)
- [Gitleaks Scanner Action](https://github.com/marketplace/actions/gitleaks-scanner) (unofficial)

## Azure Pipelines

In this section, you can find example recipes for Azure Pipelines.

> **NOTE**
>
> Official Azure Pipelines extension for Gitleaks does not exist.

### Option 1 - Gitleaks with unofficial Azure Pipelines extension

Currently, there is only one Azure Pipelines extension for Gitleaks under [Visual Studio Marketplace](https://marketplace.visualstudio.com), and it is an open-source project licensed under [MIT License](https://opensource.org/licenses/MIT).

> **NOTE**
>
> Very often, unofficial Tasks are community-powered efforts and vary code maintenance and support approaches.

[Gitleaks by Foxholenl](https://marketplace.visualstudio.com/items?itemName=Foxholenl.Gitleaks) has been selected for this Pipeline example.

Task automatically uploads SARIF report to Pipeline Artifacts, and the result can be visualized using the [SARIF SAST Scans Tab](https://marketplace.visualstudio.com/items?itemName=sariftools.scans) extension.

{% raw %}

```yaml
trigger:
- main

pr:
- main

pool:
  vmImage: ubuntu-latest

steps:
- checkout: self
  displayName: Checkout repo

- task: Gitleaks@2
  displayName: Run Gitleaks
  inputs:
    configtype: default
    scanmode: all
    verbose: true
```

{% endraw %}

### Option 2 - Gitleaks as command line

You can run Gitleaks natively as a command line without using any Azure Pipelines extension. In this scenario, you have complete control, but you are responsible for code maintenance based on Gitleaks breaking changes in future releases. Moreover, you have to setup Gitleaks on the agent as well.

The following example contains the setup step that downloads and installs the latest version of the Gitleaks from the official repository and the step with Gitleaks execution to detect secrets in the code.

{% raw %}

```yaml
trigger:
- main

pr:
- main

pool:
  vmImage: ubuntu-latest

steps:
- checkout: self
  displayName: Checkout repo

- script: |
    curl -sSLO $(curl -sSL https://api.github.com/repos/zricethezav/gitleaks/releases/latest | jq --compact-output --raw-output '.assets[] | select( .name | contains("linux_x64") ) | select( .name | endswith(".tar.gz") ).browser_download_url')
    filename=$(find . -maxdepth 1 -type f -regex '^.*\/*linux_x64*\.tar.gz')
    tar -xf "${filename}" gitleaks
    mv -f gitleaks /usr/local/bin/
    rm -f "${filename}"
  displayName: Setup Gitleaks

- script: |
    set +e
    report_name="gitleaks-report"
    report_format="sarif"
    command="gitleaks detect --redact --verbose --report-format=${report_format} --report-path=$(Pipeline.Workspace)/${report_name}.${report_format} --log-level debug"
    echo "Running Gitleaks $(gitleaks version)"
    echo "##[command]${command}"
    OUTPUT=$(eval "${command}")
    exitcode=$?

    echo "##vso[task.setvariable variable=gitleaks_command]${command}"
    echo "##vso[task.setvariable variable=gitleaks_exitcode]${exitcode}"

    if [ ${exitcode} -eq 0 ]; then
      GITLEAKS_RESULT="SUCCESS! Your code is good to go"
      echo "##[command]${GITLEAKS_RESULT}"
    elif [ ${exitcode} -eq 1 ]; then
      GITLEAKS_RESULT="STOP! Gitleaks encountered leaks or error"
      echo "##vso[task.logissue type=error]${GITLEAKS_RESULT}"
    else
      echo "##vso[task.logissue type=error]Gitleaks unknown error"
      exit ${exitcode}
    fi

    echo "${OUTPUT}"

    echo "# ${GITLEAKS_RESULT}" >$(Pipeline.Workspace)/summary.md
    echo '' >>$(Pipeline.Workspace)/summary.md
    echo '```json' >>$(Pipeline.Workspace)/summary.md
    echo "${OUTPUT}" >>$(Pipeline.Workspace)/summary.md
    echo '```' >>$(Pipeline.Workspace)/summary.md
    cat $(Pipeline.Workspace)/summary.md
    echo "##vso[task.uploadsummary]$(Pipeline.Workspace)/summary.md"

    echo "##vso[task.setvariable variable=gitleaks_report]$(Pipeline.Workspace)/${report_name}.${report_format}"

    exit ${exitcode}
  name: gitleaks
  displayName: Run Gitleaks

- task: PublishPipelineArtifact@1
  displayName: Upload SARIF report to Pipeline Artifacts
  condition: and(always(), eq(variables.gitleaks_exitcode, '1'))
  inputs:
    targetPath: $(gitleaks_report)
    artifact: CodeAnalysisLogs
    publishLocation: pipeline
```

{% endraw %}

### Azure DevOps References

- [Gitleaks - Azure Pipelines Extension](https://marketplace.visualstudio.com/items?itemName=Foxholenl.Gitleaks) (unofficial)
- [SARIF SAST Scans Tab](https://marketplace.visualstudio.com/items?itemName=sariftools.scans)

## References

- [Gitleaks](https://github.com/zricethezav/gitleaks)
