# Simplifying Configuration

## Demo

### Simple Deployment

#### YAML

```bash
kubectl get all -A

kubectl apply -f simple/nginx.yaml

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default

kubectl delete -f simple/nginx.yaml

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default
```

#### CUE

```bash
cue import -l 'kind' -l 'metadata.name' -p demo -f simple/nginx.yaml

cue export simple/nginx.cue

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' simple/nginx.cue

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' simple/nginx.cue | kubectl apply -f -

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' simple/nginx.cue | kubectl delete -f -

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default
```

### Pulling Up Configuration

```bash
cue mod init

cat >demo.cue <<EOF
package demo

Namespace: [Name=string]: {
	apiVersion: "v1"
	kind:       "Namespace"
	metadata: {
		name: Name
		labels: name: Name
	}
}

Deployment: [Name=string]: {
	apiVersion: "apps/v1"
	kind:       "Deployment"
	metadata: {
		name: Name
		labels: app: Name
	}
	spec: {
		selector: matchLabels: app: Name
		template: metadata: labels: app: Name
	}
}
EOF

cue trim -s ./simple

cue export ./simple

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' ./simple | kubectl apply -f -

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' ./simple | kubectl delete -f -

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default
```

### Multiple Configurations

```bash
cue import -l 'kind' -l 'metadata.name' -p demo -f ./multi

cue trim -s ./multi

cat > multi/multi.cue <<EOF
package demo

Deployment: [_]: spec: template: spec: containers: [{
	name:  "nginx"
	image: "nginx"
	ports: [{
		containerPort: 80
	}]
}]
EOF

cue trim -s ./multi

cue export ./multi

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' ./multi | kubectl apply -f -

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' ./multi | kubectl delete -f -

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default
```

### Tiered Configuration

```bash
cue import -l 'kind' -l 'metadata.name' -p demo -f ./org/**/*.yaml

cue trim -s ./org/*/*/

cat > org/org.cue <<EOF
package demo

Deployment: [_]: spec: template: spec: containers: [{
	name:  "nginx"
	image: "nginx"
	ports: [{
		containerPort: 80
	}]
}]
EOF

cue trim -s ./org/*/*/

cat >> org/org.cue <<EOF

_org: string | *"org"

_dept: string | *"none"

_org_dept: "\(_org)-\(_dept)"

_ns: "ns-\(_org_dept)"

_app: "app-\(_org_dept)"

_env: *"scratch" | "dev" | "qa" | "prod"

Namespace: "\(_ns)": {}

Namespace: [_]: metadata: name: _ns

Deployment: "\(_app)": {}

Deployment: [_]: metadata: namespace: _ns

Deployment: [_]: metadata: name: _app

Namespace: [_]: metadata: labels: environment: _env
Deployment: [_]: {
	metadata: labels: environment: _env
	spec: {
		selector: matchLabels: environment: _env
		template: metadata: labels: environment: _env
	}
}
EOF

# This will throw some errors, but it highlights what we need to do
cue trim -s ./org/*/*/

cat >org/dept-a/dept-a.cue <<EOF
package demo

_dept: "dept-a"
EOF

cat >org/dept-b/dept-b.cue <<EOF
package demo

_dept: "dept-b"
EOF

cue trim -s ./org/*/*/

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' ./org/dept-a/dev | kubectl apply -f -

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' ./org/dept-a/dev | kubectl delete -f -

for d in dept-a dept-b; do
  for e in dev prod; do
    cat >>org/$d/$e/*.cue <<EOF
_env: "$e"
EOF
  done
done

cue trim -s ./org/*/*/

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' ./org/dept-a/dev | kubectl apply -f -

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' ./org/dept-a/dev | kubectl delete -f -

for d in dept-a dept-b; do
  for e in dev prod; do
    cat >org/$d/$e/*.cue <<EOF
package demo

_env: "$e"
EOF
  done
done

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' ./org/dept-a/dev | kubectl apply -f -

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default

kubectl describe deployment app-org-dept-a -n ns-org-dept-a

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' ./org/dept-a/dev | kubectl delete -f -
```

## CUE Scripting

```bash
cat >demo_tool.cue <<EOF
package demo

import (
    "encoding/json"
    "tool/cli"
)

command: demo: cli.Print & {
    out: {
        apiVersion: "v1"
        kind: "List"
        items: [for x in [Namespace, Deployment] for y in x {y}]
    }
    text: json.Marshal(out)
}
EOF

cue cmd demo ./org/dept-a/dev

cue cmd demo ./org/dept-a/dev | jq

cue cmd demo ./org/dept-a/dev | kubectl apply -f -

kubectl get all -A --field-selector metadata.namespace!=kube-system,metadata.namespace!=default

cue cmd demo ./org/dept-a/dev | kubectl delete -f -
```

## Random Bits

### Import YAML

```bash
cue import -l 'kind' -l 'metadata.name' -l '"k8s"' -p demo --dryrun -f -o ./simple/nginx-k8s.cue ./simple
```

### Export K8S Configuration

```bash
cue export -e Namespace ./simple

cue export -e '[Namespace, Deployment]' ./simple

cue export -e '[for x in [Namespace, Deployment] {x}]' ./simple

cue export -e '[for x in [Namespace, Deployment] for y in x {y}]' ./simple

cue export -e '{"apiVersion": "v1", "kind": "List", "items": [for x in [Namespace, Deployment] for y in x {y}]}' ./simple
```

## References

* <https://cuelang.org/>
* <https://github.com/cue-lang/cue>
* <https://cuetorials.com/>
* [A Practical guide to CUE: patterns of everyday use](https://fosdem.org/2022/schedule/event/cue_pratical_guide/)
	* <https://github.com/cue-examples/fosdem2022>
* <https://engineering.mercari.com/en/blog/entry/20220127-kubernetes-configuration-management-with-cue/>
* <https://engineering.mercari.com/en/blog/entry/20220122-adventures-of-using-cue-at-scale/>
* <https://www.alibabacloud.com/blog/configure-a-new-capability-for-kubernetes-paas-in-20-minutes_597277>
* <https://bitfieldconsulting.com/golang/cuelang-exciting>
* <https://github.com/thesecuresoftwarefactory/ssf>
