# Helm Charts

This repository contains Helm charts for the Sendent applications.

## Charts

| Chart                              | Description                                                               |
|------------------------------------|---------------------------------------------------------------------------|
| [outlook-addin](./outlook-addin/)  | Sendent Outlook Add-in - Nextcloud integration for Microsoft Outlook      |
| [ms-teams-addin](./msteats-addin/) | Sendent Microsoft Teams Add-in - Nextcloud integration of Microsoft Teams |

## Usage

```bash
helm install <release-name> ./<chart-name> -f your-values.yaml
```

See each chart's own README for configuration details.
