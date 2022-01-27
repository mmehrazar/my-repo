# my-repo

Flux to Tekton


$ kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.18.1/release.yaml
Install Tekton Triggers.

$ kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/previous/v0.10.1/release.yaml
$ flux bootstrap github --owner=$GITHUB_USER --repository=fleet-infra --branch=main --path=staging-cluster --personal

$ kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.18.1/release.yaml

service/tekton-pipelines-webhook created
$ kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/previous/v0.10.1/release.yaml
podsecuritypolicy.policy/tekton-triggers created
<snip>
deployment.apps/tekton-triggers-webhook created

  
  Configuring Tekton

The event documentation could do with a JSON example for folks who don’t read Go natively, but the requests look like this:

POST / HTTP/1.1
Host: example.com
Accept-Encoding: gzip
Content-Length: 452
Content-Type: application/json
Gotk-Component: source-controller
User-Agent: Go-http-client/1.1
With a body that looks like this:

{
  "involvedObject": {
    "kind":"GitRepository",
    "namespace":"flux-system",
    "name":"flux-system",
    "uid":"cc4d0095-83f4-4f08-98f2-d2e9f3731fb9",
    "apiVersion":"source.toolkit.fluxcd.io/v1beta1",
    "resourceVersion":"56921",
  },
  "severity":"info",
  "timestamp":"2020-11-27T15:52:21Z",
  "message":"Fetched revision: main/731f7eaddfb6af01cb2173e18f0f75b0ba780ef1",
  "reason":"info",
  "reportingController":"source-controller",
  "reportingInstance":"source-controller-7c7b47f5f-8bhrp",
}
This event is emitted when the Source Controller detects a change to the watched GitRepository.

Tekton Pipeline
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: flux-demo-pipeline
spec:
  params:
    - name: message
      description: incoming notification message
      type: string
    - name: repository
      description: The name of the notification GitRepository
      type: string
    - name: namespace
      description: The namespace of the notification GitRepository
      type: string
  tasks:
    - name: task-echo-message
      taskSpec:
        params:
          - name: message
            type: string
          - name: repository
            type: string
          - name: namespace
            type: string
        steps:
          - name: echo-pipeline
            image: alpine
            command:
              - echo
            args:
              - "notification from $(params.namespace)/$(params.repository)"
          - name: echo-message
            image: alpine
            command:
              - echo
            args:
              - "$(params.message)"
      params:
        - name: message
          value: $(params.message)
        - name: repository
          value: $(params.repository)
        - name: namespace
          value: $(params.namespace)
This is a really simple Pipeline that accepts a param with a message to be echoed to the logs, and the name of the affected GitRepository CR, you could parse out the branch/revision and fetch the GitRepository to do more work, or perhaps these could be provided in the Metadata field of the event, which would definitely simplify the process.

Triggering the Pipeline from an EventListener
I created a simple EventListener to receive the notifications.

apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: flux-event-listener
spec:
  serviceAccountName: flux-notifications-sa
  triggers:
    - name: flux-notification-trigger
      interceptors:
        - cel:
            filter: header.canonical('Gotk-Component') == 'source-controller' &&
              body.involvedObject.kind == 'GitRepository'
      template:
        name: flux-notification-template
      bindings:
        - ref: flux-notification-binding
This is matching on notifications from the Source Controller.

apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerTemplate
metadata:
  name: flux-notification-template
spec:
  params:
    - name: message
      description: message
    - name: repository
      description: the notifying repository
    - name: namespace
      description: the namespace for the notifying repository
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        annotations:
        name: flux-notification-run-$(uid)
      spec:
        params:
          - name: message
            value: $(tt.params.message)
          - name: repository
            value: $(tt.params.repository)
          - name: namespace
            value: $(tt.params.namespace)
        pipelineRef:
          name: flux-demo-pipeline
This template is creating a PipelineRun with a generateName with the three parameters that the Pipeline requires from the TriggerBinding below:.

apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: flux-notification-binding
spec:
  params:
    - name: message
      value: $(body.message)
    - name: repository
      value: $(body.involvedObject.name)
    - name: namespace
      value: $(body.involvedObject.namespace)
There is some permission configuration to allow an EventListener to run in a namespace, this is a long set of configuration items, which while important for getting the EventListener to work isn’t really connected to setting up the notifications.

Configuring Flux
First off, this needs a random token secret for the receiver to generate a secret with a token:

#!/bin/sh
# Put this in generate_secret.sh and chmod +x
TOKEN=$(head -c 12 /dev/urandom | shasum | cut -d ' ' -f1)
kubectl create secret generic $1 -n flux-system \
  --from-literal=token=$TOKEN
Creating a secret is as simple as:

$ ./generate_secret.sh generic-receiver-token
secret/generic-receiver-token created
Creating a Receiver
Toolkit components send HTTP requests to an HTTP endpoint that is part of the notification-controller, so have to configure a receiver for this:

apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Receiver
metadata:
  name: generic-receiver
  namespace: flux-system
spec:
  type: generic
  resources:
    - kind: GitRepository
      name: flux-system
      namespace: flux-system
  secretRef:
    name: generic-receiver-token
Also need a Provider, this is the element that defines how messages transmitted, there are providers for Slack and GitHub, but for this case, use a generic webhook provider which just sends an HTTP body.

apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Provider
metadata:
  name: tekton-el-provider
  namespace: flux-system  
spec:
  type: generic
  address: http://el-flux-event-listener.default.svc.cluster.local:8080/
This is using standard Kubernetes DNS resolution to direct events to the Tekton EventListener defined above, when Tekton Triggers creates an EventListener process to receive HTTP requests, it creates a Service and Deployment with the prefix el- and the name of the EventListener object, and place it in the default namespace.

Next, there’s an Alert which connects the notification to the Provider.

apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Alert
metadata:
  name: tekton-alert
  namespace: flux-system
spec:
  providerRef:
    name: tekton-el-provider
  eventSeverity: info
  eventSources:
    - kind: GitRepository
      name: flux-system
      namespace: flux-system
This configures an alert for info-level notifications for the GitRepository object that’s created during the installation of flux, to be sent via the Provider defined above.

Modifying a file in the repository pointed to by the flux-system GitRepository triggers a notification.

You can see the logs of the notification being triggered:

$ kubectl logs deploy/notification-controller -n flux-system
{"level":"info","ts":"2020-12-05T13:15:53.235Z","logger":"event-server","msg":"Dispatching event","object":"flux-system/flux-system","kind":"GitRepository","message":"Fetched revision: main/6fca464c6c124156b9cb3913c229b59148323703"}
{"level":"info","ts":"2020-12-05T13:15:56.408Z","logger":"event-server","msg":"Discarding event, no alerts found for the involved object","object":"flux-system/flux-system","kind":"Kustomization"}
And in the logs of the EventListener, the EventListener is logging out that it’s creating a new resource called flux-notification-run-rqswx.

$ kubectl logs deploy/el-flux-event-listener
{"level":"info","ts":"2020-12-05T13:15:53.265Z","logger":"eventlistener","caller":"sink/sink.go:237","msg":"ResolvedParams : [{Name:message Value:Fetched revision: main/6fca464c6c124156b9cb3913c229b59148323703} {Name:repository Value:flux-system} {Name:namespace Value:flux-system}]","knative.dev/controller":"eventlistener","/triggers-eventid":"wvd9c","/trigger":"flux-notification-trigger"}
{"level":"info","ts":"2020-12-05T13:15:53.268Z","logger":"eventlistener","caller":"resources/create.go:95","msg":"Generating resource: kind: &APIResource{Name:pipelineruns,Namespaced:true,Kind:PipelineRun,Verbs:[delete deletecollection get list patch create update watch],ShortNames:[pr prs],SingularName:pipelinerun,Categories:[tekton tekton-pipelines],Group:tekton.dev,Version:v1beta1,StorageVersionHash:RcAKAgPYYoo=,}, name: flux-notification-run-rqswx","knative.dev/controller":"eventlistener"}
{"level":"info","ts":"2020-12-05T13:15:53.268Z","logger":"eventlistener","caller":"resources/create.go:103","msg":"For event ID \"wvd9c\" creating resource tekton.dev/v1beta1, Resource=pipelineruns","knative.dev/controller":"eventlistener"}
And finally, I get to see the message from the event:

$ tkn pipelinerun logs --last
[task-echo-message : echo-pipeline] notification from flux-system/flux-system

[task-echo-message : echo-message] Fetched revision: main/6fca464c6c124156b9cb3913c229b59148323703
Right now, the Notifications Controller is functional, and hopefully it will become easier to integrate components.
