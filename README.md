### Network Traffic Rules
#### Grafana Cloud
To establish an SSH connection to Grafana Cloud, the PDC agent must run on a network that allows internet egress to the following endpoints: 

```
private-datasource-connect-prod-eu-west-2.grafana.net:22
private-datasource-connect-api-prod-eu-west-2.grafana.net:443
``` 

#### Dashboards
In order to access the local dashboards of the Infocare deplyment, the network must allow ingress traffic towards the infocare-container.


### PDC Token
In order for the Private Datasource Connect to run, a valid token must be supplied. This token needs to be generated when a new PDC network is defined in Grafana Cloud.
### Create Kubernetes Secrets
First create a k8s secret containing the PDC information.
```shell
kubectl create secret generic grafana-pdc-agent \
 --from-literal="token=<INFOCARE_PDC_TOKEN>" \
 --from-literal="hosted-grafana-id=850040" \
 --from-literal="cluster=prod-eu-west-2"
```

Next create a k8s secret containing the database monitor user password
```shell
kubectl create secret generic ic-db-user \
    --from-literal=IC_DB_USER=<infocare database user name>
    --from-literal=IC_DB_PASS=<infocare database user password>
```
*Note:*  
You must use single quotes '' to escape special characters such as $, \, *, =, and ! in your strings. If you don't, your shell will interpret these characters.

### Configuration
#### Dashboards NodePort
In order to access the dashboards a NodePort is created. The default value for this NodePort is 32000.  
If you wish to change this you will need to change it in the *dashboards.yml* file.

#### Exporters
Create a configMap containing information for the exporter
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ic-exporters-config
data:
  exporter1.db_host: "192.168.122.138"
  exporter1.db_port: "25000"
  exporter1.db_name: sample
```

### (Optional) Add Aditional Exporters
In case you want to monitor more than a single database you will need to define extra exporters and extra ConfigMaps to be used by these exporters.

Add to the exporters ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: extra-config
data:
  ...
  exporter2.db_host: "192.168.122.50"
  exporter2.db_port: "25000"
  exporter2.db_name: another
```

To add an extra exporter add an extra container definition.
```yaml
apiVersion: v1
kind: Pod
...
spec:
  containers:
  ...
  - name: extra-exporter-container
  ...
    ports:
      - containerPort: 9954
        name: extra-port-name
    env:
    - name: DB_HOST2
      valueFrom:
        configMapKeyRef:
          name: ic-exporters-config
          key: exporter2.db_host
    - name: DB2_PORT2
      valueFrom:
        configMapKeyRef:
          name: ic-exporters-config
          key: exporter2.db_port
    - name: DB2_NAME2
      valueFrom:
        configMapKeyRef:
          name: ic-exporters-config
          key: exporter2.db_name
    ...
    command:
      - /opt/run
      - $(DB_HOST2)
      - $(DB_PORT2)
      - $(DB_NAME2)
      - $(DB_USER)
      - $(DB_USER)
      - $(DB_PASS)
```

Add an extra port definition to the exporters service.
```yaml
apiVersion: v1
kind: Service
...
  ports:
  ...
  - name: extra-port-name
    protocol: TCP
    port: 9954
    targetPort: extra-port-name

```

### Deploy
Assuming the ConfigMap file for the exporters is named *exporters.yml*.
```shell
kubectl apply -f exporters-config.yml \
  -f prometheus.yml \
  -f db2-exporters.yml \
  -f remote-agent.yml \
  -f dashboards.yml \
  -f pdc-agent.yml
```