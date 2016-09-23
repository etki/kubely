# Huge draft warning

There is no project behind this readme. The whole document is just an idea i
wish to implement one day.

## Kubely

Kubely is a rendering and application CLI tool for complex Kubernetes 
deployment configurations: it takes bunch of configuration templates, 
renders them using provided context, and then applies them in specific order.
Kubely may help you in following cases:

- Your deployments have common context that forces you to change logs of files
even because of small changes
- Your deployments contain sensitive data (secrets, yeap) that you want to 
stay out of configuration repository
- Your deployments have repeated configurations, (e.g. you're imitating petset 
using several deployments)

Kubely allows you to store configuration templates rather than built 
configuration and easily modify complex setups without much fuzz.

### Classic quickie

```bash
$ echo <<EOF > src/project-name/default/context.yml
pod_port: 8080
labels:
  app: nginx
  deployment: green
EOF

$ cat <<EOF > src/project-name/default/service/http.yml
kind: Service
apiVersion: v1
spec:
  selector:
    {% for label, value in labels %}
    {{ label }}: {{ value }}
    {% endfor %}
  ports:
    - name: HTTP
      protocol: TCP
      port: 80
      targetPort: {{ pod_port }}
EOF

$ kubely render -p src/project-name -o build/project-name
Rendering src/project-name/default/service/http.yml
Finished without errors

$ cat build/project-name/default/service/http.yml

kind: Service
apiVersion: v1
metadata:
  name: http
  namespace: project-name
spec:
  selector:
    app: nginx
    deployment: green
  ports:
    - name: HTTP
      protocol: TCP
      port: 80
      targetPort: 8080

$ kubely apply -p src/project-name -o build/project-name
Rendering src/project-name/default/service/http.yml
Applying build/project-name/default/service/http.yml
Finished without errors
```

### Projects and project structure

Kubely divides cluster into projects, where every project is meant to occupy a
namespace (not necessarily, but strongly recommended) and form an indivisible
deployment, e.g. `web-ui`, `backend`, `video-streaming` or something else.

Every project is represented as several possible environments, each of which, 
in turn, occupies another directory and is a set of configuration templates.
Environments may take any name that doesn't have any restrictions, except for
`default` that is used as parent for all other environments.

Following structure represents simple project:

```
infrastructure:
  configuration.yml # optional
  context.yml # optional
  default: # used as source for all other environments
    context.yml # optional, overrides conflicting keys in ../context.yml
    deployment:
      aerospike.yml
      amqp.yml
  production:
    context.yml # optional, overrides conflicting keys in previous contexts
    deployment:
      aerospike.yml # overrides aerospike.yml from default environment
      .wh.amqp.yml # removes amqp.yml from final build
```

### Project configuration

Every project is analyzed for `configuration.yml` file presence. This file may
alter kubely behavior using following options:

```yml
# By default project name is derived from it's directory
project_name: something-secret

# List of parent contexts for current project
parent_context:
  # You may have parent context for several projects in one repo
  - path: ../root-context.yml
  # You may also specify optional context that will be included only if present
  - path: ~/custom-kubely-config.yml
    required: false
    
# Namespace configuration file may be created automatically
create_namespace_declaration: true

# By default project namespace is automatically injected in all configurations
inject_namespace: true

# Whether to stop processing if an error has been encountered
halt_on_error: true

# You can define custom directory application order
application_order: 
  - namespace
  - service
  - secret
  - deployment

# Classic glob-pattern exclusion list
ignore:
  - template
```

### Context sources

Kubely retrieves context in following pattern:

- Sources in `configuration.yml` are parsed and merged
- `%project%/context.yml`, `%project%/default/context.yml` and 
`%project%/%environment%/context.yml` are parsed and merged in the same way
- Environment variables with prefix `KUBELY_UDC_` are analyzed, then their 
content is parsed (algorithm will be explained in a second) and merged in
existing context
- Values specified via `--user-defined-context` are parsed and merged in the 
same way

Because last two techniques may specify values only in "plain" mode, they use
special `property.path={"json-encoded": "value"}` pattern, where text before
`=` symbol denotes property path in context, and text after `=` is treated as 
json string.

### Iterated templates

Above all specified, kubely has support for iterated templates - that's a
template that is iterated using variable in provided context and may produce
several nearly identical configuration files. The original case for this was
petset emulation using deployments.

Following files demonstrate iterated template example:

deployment/reverse-proxy.yml:

```yml
kind: IteratedTemplate
apiVersion: kubely/v1
spec:
  propertyPath: services.application
  templatePath: template/application-reverse-proxy.yml
  targetPath: deployment/{{ cursor.id }}-reverse-proxy.yml
```

template/application-reverse-proxy.yml

```yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: {{ cursor.id }}-reverse-proxy
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: {{ cursor.id }}-reverse-proxy
    spec:
      containers:
      - name: reverse-proxy
        image: gcr.io/project/reverse-proxy
        ports:
          - containerPort: 80
        env:
          - name: APPLICATION_HOST
            value: {{ cursor.id }}.{{ cursor.namespace }}.svc.cluster.local
            
```

In the case of iterated template context gets cloned, then it is populated with
`cursor` and `loop` variables.

### Application order

Kubely applies configuration in following order by default:

- namespace/*
- secret/*
- config-map/*
- configmap/*
- volume/*
- pvc/*
- service/*
- ingress/*
- daemon-set/*
- daemonset/*
- daemon/*
- deployment/*
- replica-set/*
- replicaset/*
- pod/*
- job/*
- everything not applied before
