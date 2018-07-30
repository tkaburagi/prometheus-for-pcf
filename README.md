# outside-pcfbosh-prometheus-bosh
Sample Enviroment. This doc is for deloying prometheus-boshrelease to monitor PCF.
![](https://github.com/tkaburagi/outside-pcfbosh-prometheus-bosh/blob/master/diagram.png)

## Creating BOSH ENV on GCP
Following [instruction](https://bosh.io/docs/init-google/).
```console
bosh2 create-env bosh-deployment/bosh.yml \
    --state=state.json \
    --vars-store=creds.yml \
    -o bosh-deployment/gcp/cpi.yml \
    -v director_name=d-bosh \
    -v internal_cidr=10.0.0.0/24 \
    -v internal_gw=10.0.0.1 \
    -v internal_ip=10.0.0.6 \
    --var-file gcp_credentials_json=<<KEY_JSON>> \
    -v project_id=<<GCP_PROJECT_ID>> \
    -v zone=<<ZONE>> \
    -v tags=[d-bosh] \
    -v network=<<GCP_VPC>> \
    -v subnetwork=<<GCP_VPC_SUBNET>>
```

Logging into BOSH ENV. `cred.yml` should be stored the directory which is set as `--vars-store`.
```console
bosh2 alias-env gcpbosh -e 10.0.0.6 --ca-cert <(bosh2 int /home/tkaburagi/creds.yml --path /director_ssl/ca)
export BOSH_CLIENT=admin
export BOSH_CLIENT_SECRET=`bosh2 int /home/tkaburagi/creds.yml --path /admin_password`
bosh2 -e bosh env
Using environment '10.0.0.6' as client 'admin'

Name      d-bosh
UUID      c672a233-2ab8-4710-ba7d-1845e223419b
Version   266.7.0 (00000000)
CPI       google_cpi
Features  compiled_package_cache: disabled
          config_server: disabled
          dns: disabled
          snapshots: disabled
User      admin

Succeeded
```


## Updating Cloud Config
```console
bosh2 -e bosh-1 update-cloud-config
bosh2 -e bosh-1 upload-stemcel
```

## Generating UAA Clients for Exporters
```console
uaac target
uaac token client
uaac client add firehose_exporter
uaac client add cf_exporter
uaac target
uaac token client
uaac client add bosh_exporter
```
https://github.com/bosh-prometheus/node-exporter-boshrelease

## Deploying Prometheus
```console
bosh2 -e bosh -d prometheus deploy manifests/prometheus.yml \
  --vars-store tmp/deployment-vars.yml \
  -o manifests/operators/monitor-bosh.yml \
  -o manifests/operators/monitor-cf.yml \
  -o manifests/operators/monitor-node.yml \
  -o pcf-colocate-firehose_exporter.yml \
  -o pcf-local-exporter.yml \
  -v bosh_url=\
  -v bosh_username= \
  -v bosh_password=u \
  --var-file bosh_ca_cert= \
  -v metrics_environment=pcflab \
  -v metron_deployment_name=cf \
  -v system_domain= \
  -v uaa_bosh_exporter_client_secret=prometheus-client-secret \
  -v uaa_clients_cf_exporter_secret=prometheus-client-secret \
  -v uaa_clients_firehose_exporter_secret=prometheus-client-secret \
  -v traffic_controller_external_port=443 \
  -v skip_ssl_verify=true
```
