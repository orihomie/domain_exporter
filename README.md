# domain_exporter

Exports the expiration time of your domains as prometheus metrics.

#### Environment variables

- `DOMAIN_EXPORTER_URL_PREFIX` - use when HTTP endpoint served with a prefix, e.g.:

  For this endpoint `http://example.org/exporters/domains` set to `/exporters/domains`.

  Not really required since useful only to prevent breaking human-oriented links.

  Defaults to empty string.

## Configuration

On the prometheus settings, add the `domain_exporter` prober:

```yaml
- job_name: domain
  metrics_path: /probe
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: localhost:9222 # domain_exporter address
  static_configs:
    - targets:
      - carlosbecker.com
      - carinebecker.com
      - watchub.pw
```

It works more or less like prometheus's
[blackbox_exporter](https://github.com/prometheus/blackbox_exporter).

Alerting rules examples can be found on the
[_examples](https://github.com/caarlos0/domain_exporter/tree/main/_examples)
folder.

You can configure `domain_exporter` to always export metrics for specific domains.
Create configuration file:
```yaml
domains:
- google.com
- reddit.com
```
And pass file path as agrument to `domain_exporter`:
```
domain_exporter --config domains.yaml
```

## Install

**helm**:

```sh
helm install domain-exporter .
```

For sending alerts to telegram you'll need to get a littl bit tricky:

1. Create telegram bot and retrieve its token from [BotFather](https://t.me/BotFather), then create secret in the same namespace with alertmanager deployment. <br>
Let's suppose your release would be called `domainexporter`:
```secret-sample.yaml
apiVersion: v1
kind: Secret
metadata:
  name: domainexporter
  labels:
    helm.sh/chart: domainexporter-0.1.0
    app.kubernetes.io/name: domainexporter
    app.kubernetes.io/instance: domainexporter
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
    release: kps
type: Opaque
data:
  token: your_token_base64_formatted
```

2. update default kube-prom-stack or whatever alertmanager config with the following:

```values.override.yaml
alertmanager:

  tplConfig: true
  templateFiles:
     templates.tmpl: |-
        {{ define "cluster" }}{{ .ExternalURL | reReplaceAll ".*alertmanager\\.(.*)" "$1" }}{{ end }}

        {{ define "slack.myorg.text" }}
        {{- $root := . -}}
        {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }} - `{{ .Labels.severity }}`
          *Cluster:* {{ template "cluster" $root }}
          *Description:* {{ .Annotations.description }}
          *Graph:* <{{ .GeneratorURL }}|:chart_with_upwards_trend:>
          *Runbook:* <{{ .Annotations.runbook }}|:spiral_note_pad:>
          *Details:*
            {{ range .Labels.SortedPairs }} - *{{ .Name }}:* `{{ .Value }}`
            {{ end }}
        {{ end }}
        {{ end }}

        {{ define "telegram.text" }}
        {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Labels:*
            {{ range .Labels.SortedPairs }}
              {{ .Name }} "{{ .Value }}"
            {{ end }}
          *Details:*
            {{ range .Labels.SortedPairs }}
              {{ .Name }} "{{ .Value }}"
            {{ end }}
        {{ end }}
        {{ end }}
```

Follow [this](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#get-helm-repository-info) instructions first.
And then apply that changes:
> helm upgrade -f values.overrided.yaml your_release_name prometheus-community/kube-prometheus-stack

Now, I recommend you to delete alertmanager's pod for it to get updates.

**homebrew**:

```sh
brew install caarlos0/tap/domain_exporter
```

**docker**:

```sh
docker run --rm -p 9222:9222 caarlos0/domain_exporter
```

**apt**:

```sh
echo 'deb [trusted=yes] https://repo.caarlos0.dev/apt/ /' | sudo tee /etc/apt/sources.list.d/caarlos0.list
sudo apt update
sudo apt install domain_exporter
```

**yum**:

```sh
echo '[caarlos0]
name=caarlos0
baseurl=https://repo.caarlos0.dev/yum/
enabled=1
gpgcheck=0' | sudo tee /etc/yum.repos.d/caarlos0.repo
sudo yum install domain_exporter
```

**deb/rpm/apk**:

Download the `.apk`, `.deb` or `.rpm` from the [releases page][releases] and install with the appropriate commands.

**manually**:

Download the pre-compiled binaries from the [releases page][releases] or clone the repo build from source.

[releases]: https://github.com/caarlos0/domain_exporter/releases

## Stargazers over time

[![Stargazers over time](https://starchart.cc/caarlos0/domain_exporter.svg)](https://starchart.cc/caarlos0/domain_exporter)
