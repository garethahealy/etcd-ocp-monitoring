[![Build Status](https://travis-ci.org/garethahealy/etcd-ocp-monitoring.svg?branch=master)](https://travis-ci.org/garethahealy/etcd-ocp-monitoring)
[![License](https://img.shields.io/hexpm/l/plug.svg?maxAge=2592000)]()

# etcd-ocp-monitoring 
## Disclaimer
**Supportability:**
*If you are a RedHat customer; it is __suggested__ you raise a support ticket to clarify the supportability of the below components.
The below are the views of my own, and not representative of RedHat nor of any communities mentioned.*
- Hawkular OpenShift Agent is only supported [upstream](http://www.hawkular.org/community/docs/getting-involved/)
- Docker container image 'ocp-external-service-onramp' [best endeavours](https://github.com/garethahealy/ocp-external-service-onramp/issues)
- etcd is supported as part of an OCP subscription by [RedHat support](https://access.redhat.com/support/cases), as long as, it has been installed and configured as per the OCP documentation and not tampered with.
- Hawkular Grafana DataSource is only supported [upstream](https://github.com/hawkular/hawkular-grafana-datasource/issues)

## Preface
This blog post is a continuation of [FIS2.0 OCP Monitoring via Hawkular OpenShift Agent](https://github.com/garethahealy/fis2-ocp-monitoring). 
Before continuing; it is expected you have deployed the Hawkular OpenShift Agent and monitoring an example application. 

### Scenario
etcd is a key-value store and a core component of OCP.


A core component of OCP is etcd, It is a key-value store of information which, by default, exposes a [Prometheus enabled metrics endpoint](https://github.com/coreos/etcd/blob/master/Documentation/metrics.md).
As the Hawkular OpenShift Agent is able to communicate with Promethues endpoints, the configuration should be quite easy to piece together.

### Solution
todo

## Example Deployment
### Find and validate etcd client certificates
In OCP, etcd is protected by certificates which are located on the master node. If you are unsure where these are, the below command sh

    sudo su
    find / -name master-config.yaml

On the CDK, they are in:

    cd /var/lib/origin/openshift.local.config/master/

Check its correct:

    grep -A5 "etcdClientInfo" master-config.yaml

![alt text](https://github.com/garethahealy/etcd-ocp-monitoring/raw/master/screenshots/etcd_client_info.png "ectd client info")

    curl https://10.2.2.2:4001/metrics --cacert ./ca.crt --cert ./master.etcd-client.crt --key ./master.etcd-client.key

![alt text](https://github.com/garethahealy/etcd-ocp-monitoring/raw/master/screenshots/prometheus_output.png "Prometheus metrics output")

### Add etcd client certificates to Hawkular OpenShift Agent
We want to add the certificates so they can be used by a pod:
    
    oc project openshift-infra
    
    oc secrets new etcd-client-crt master.etcd-client.crt
    oc secrets new etcd-client-key master.etcd-client.key

    oc volume rc/hawkular-openshift-agent --add --name=etcd-client-crt --type=secret --secret-name=etcd-client-crt --mount-path=/var/run/secrets/custom/etcd-client-crt
    oc volume rc/hawkular-openshift-agent --add --name=etcd-client-key --type=secret --secret-name=etcd-client-key --mount-path=/var/run/secrets/custom/etcd-client-key

And add the keys to the agent, as etcd is mutual auth protected

    oc edit configmap hawkular-openshift-agent-configuration
    
    identity:
      cert_file: /var/run/secrets/custom/etcd-client-crt/master.etcd-client.crt
      private_key_file: /var/run/secrets/custom/etcd-client-key/master.etcd-client.key

![alt text](https://github.com/garethahealy/etcd-ocp-monitoring/raw/master/screenshots/agent_yaml.png "Agent configmap")

### Deploy ectd-onramp
As the pod will be privileged, we want to segergate it from others, so it lives in its other project

    oc new-project etcd-onramp
    
Give the default SA privileged SCC and create the config:

    oc adm policy add-scc-to-user privileged system:serviceaccount:etcd-onramp:etcd-external
    oc create -f https://raw.githubusercontent.com/garethahealy/etcd-ocp-monitoring/master/ocp-template/etcd2-onramp-hawkular-agent.yaml
    oc process -f https://raw.githubusercontent.com/garethahealy/etcd-ocp-monitoring/master/ocp-template/etcd-external-service-onramp.yaml | oc create -f -
    

Below should show metrics being collected and stored for etcd

    TOD: get pod -oname -lhawkular-openshift-agent=true
    AGENT_POD=$(oc get pod -n openshift-infra | grep hawkular-openshift-agent | cut -d' ' -f1)
    oc logs -n openshift-infra -f $AGENT_POD
    
We are now ready to start visualizing with Grafana!

![alt text](https://github.com/garethahealy/etcd-ocp-monitoring/raw/master/screenshots/agent_log.png "Agent log")

## Conclusion
todo

### Whats next?
TODO:
- identity issue
- labels metrics as seperate, instead of all together as is now
