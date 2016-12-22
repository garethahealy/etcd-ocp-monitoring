[![License](https://img.shields.io/hexpm/l/plug.svg?maxAge=2592000)]()

# etcd-ocp-monitoring 
## Preface
This blog post is a continuation of [FIS2.0 OCP Monitoring via Hawkular OpenShift Agent](https://github.com/garethahealy/fis2-ocp-monitoring). 
Before continuing, it is expected that you have completed the steps in the mentioned blog post.

### Monitoring etcd
A core component of OCP is etcd, It is a key-value store of information which, by default, exposes a [Prometheus enabled metrics endpoint](https://github.com/coreos/etcd/blob/master/Documentation/metrics.md).
As the Hawkular OpenShift Agent is able to communicate with Promethues endpoints, the configuration should be quite easy to piece together.

#### etcd Certifcates
In OCP, etcd is protected by certificates which are located on the master node. If you are unsure where these are, the below command sh

    sudo su
    find / -name master-config.yaml

On the CDK, they are in:

    cd /var/lib/origin/openshift.local.config/master/

Check its correct:

    grep -A5 "etcdClientInfo" master-config.yaml
    
    curl https://10.2.2.2:4001/metrics --cacert ./ca.crt --cert ./master.etcd-client.crt --key ./master.etcd-client.key

We want to add the certificates so they can be used by a pod:
    
    oc project openshift-infra
    
    oc secrets new etcd-client-crt master.etcd-client.crt
    oc secrets new etcd-client-key master.etcd-client.key

    oc volume rc/hawkular-openshift-agent --add --name=etcd-client-crt --type=secret --secret-name=etcd-client-crt --mount-path=/var/run/secrets/custom/etcd-client-crt
    oc volume rc/hawkular-openshift-agent --add --name=etcd-client-key --type=secret --secret-name=etcd-client-key --mount-path=/var/run/secrets/custom/etcd-client-key

    oc edit configmap hawkular-openshift-agent-configuration
    
    identity:
      cert_file: /var/run/secrets/custom/etcd-client-crt/master.etcd-client.crt
      private_key_file: /var/run/secrets/custom/etcd-client-key/master.etcd-client.key

As the pod will be privileged, we want to segergate it from others, so it lives in its other project

    oc new-project etcd-onramp
    
Give the default SA privileged SCC and create the config:

    oadm policy add-scc-to-user privileged privileged system:serviceaccount:etcd-onramp:default
    oc create -f https://raw.githubusercontent.com/garethahealy/etcd-ocp-monitoring/master/ocp-template/etcd2-onramp-hawkular-agent.yaml
    oc process -f https://raw.githubusercontent.com/garethahealy/etcd-ocp-monitoring/master/ocp-template/etcd-external-service-onramp.yaml | oc create -f -
    

Below should show metrics being collected and stored for etcd

    AGENT_POD=$(oc get pod -n openshift-infra | grep hawkular-openshift-agent | cut -d' ' -f1)
    oc logs -n openshift-infra $AGENT_POD
    
Now we can view in grafana.
