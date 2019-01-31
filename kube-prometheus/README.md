# kube-prometheus

该项目的内容是用 jsonnet 生成的。

包含以下组件:

- The Prometheus Operator

- 高可用的 prometheus

- 高可用的 Alertmanager

- Prometheus node-exporter

- kube-state-metrics

- Grafana

如果想自己生成使用以下选项


安装 jb

```
go get github.com/jsonnet-bundler/jsonnet-bundler/cmd/jb
```

安装 gojsontoyaml

```
go get github.com/brancz/gojsontoyaml
```

安装 jsonnet

```
go get github.com/google/go-jsonnet/jsonnet
```

如果不使用此仓库的模板，可以从官方下载

```
mkdir my-kube-prometheus
cd  my-kube-prometheus

# 初始化一个空的 `jsonnetfile.json`
jb init  

# 安装 kube-prometheus  依赖, 创建 `vendor/` 目录 和  `jsonnetfile.lock.json` 和将依赖写入 `jsonnetfile.json` 文件
https_proxy=127.0.0.1:1080 jb install github.com/coreos/prometheus-operator/contrib/kube-prometheus/jsonnet/kube-prometheus
```


创建 example.jsonnet

```
local kp =
  (import 'kube-prometheus/kube-prometheus.libsonnet') + {
    _config+:: {
      namespace: 'monitoring',
    },
  };

{ ['00namespace-' + name]: kp.kubePrometheus[name] for name in std.objectFields(kp.kubePrometheus) } +
{ ['0prometheus-operator-' + name]: kp.prometheusOperator[name] for name in std.objectFields(kp.prometheusOperator) } +
{ ['node-exporter-' + name]: kp.nodeExporter[name] for name in std.objectFields(kp.nodeExporter) } +
{ ['kube-state-metrics-' + name]: kp.kubeStateMetrics[name] for name in std.objectFields(kp.kubeStateMetrics) } +
{ ['alertmanager-' + name]: kp.alertmanager[name] for name in std.objectFields(kp.alertmanager) } +
{ ['prometheus-' + name]: kp.prometheus[name] for name in std.objectFields(kp.prometheus) } +
{ ['prometheus-adapter-' + name]: kp.prometheusAdapter[name] for name in std.objectFields(kp.prometheusAdapter) } +
{ ['grafana-' + name]: kp.grafana[name] for name in std.objectFields(kp.grafana) }
```

编译, 将下面脚本写入 build.sh , `./build.sh example.jsonnet`

```
#!/usr/bin/env bash

# This script uses arg $1 (name of *.jsonnet file to use) to generate the manifests/*.yaml files.

set -e
set -x
# only exit with zero if all commands of the pipeline exit successfully
set -o pipefail

# Make sure to start with a clean 'manifests' dir
rm -rf manifests
mkdir manifests

                                               # optional, but we would like to generate yaml, not json
jsonnet -J vendor -m manifests "${1-example.jsonnet}" | xargs -I{} sh -c 'cat {} | gojsontoyaml > {}.yaml; rm -f {}' -- {}
```


