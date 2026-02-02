# olake kustomize examples

This repo contains base Kustomize manifests plus environment overlays for running an Olake **discover** Job and **sync** CronJob.

## Prerequisites

- A Kubernetes cluster and `kubectl` access
- `kustomize` (or `kubectl` with built-in Kustomize support)
- Namespace `olake` (manifests set `namespace: olake`)

## Layout

- `base/discover` — reusable Job manifest (discover)
- `base/sync` — reusable CronJob manifest (sync) + PVC
- `sources/logistics_db` — overlay that generates the **source** secret
- `destinations/lake` — overlay that generates the **destination** secret
- `streams/ligistics_aws_sc` — overlay that generates the **streams** configmap and wires sync names

## Discover: example run

1) Apply the source secret and discover Job:

```sh
kubectl apply -k sources/logistics_db
```

2) Wait for completion and view logs:

```sh
kubectl -n olake wait --for=condition=complete job/logistics-db-olake-source-check --timeout=300s
kubectl -n olake logs job/logistics-db-olake-source-check -c discover
```

## Sync: example run

1) Apply destination secret and the sync CronJob overlay:

```sh
kubectl apply -k destinations/lake
kubectl apply -k streams/ligistics_aws_sc
```

2) (Optional) trigger a one-off run from the CronJob:

```sh
JOB_NAME=sync-once-$(date +%s)
kubectl -n olake create job --from=cronjob/cron-logistics-db-to-lake-sync $JOB_NAME
```

3) Watch logs for the one-off job:

```sh
kubectl -n olake logs job/$JOB_NAME -c sync
```

Note: the CronJob runs on the schedule defined in `base/sync/job-sync.yaml` and uses the PVC `sync-state-pvc` (name-prefixed by the overlay).

## Reuse in a new folder

`base/sync/job-sync.yaml` uses placeholder names:

- `REPLACEME_SOURCE_SECRET`
- `REPLACEME_DEST_SECRET`
- `REPLACEME_STREAMS_CONFIGMAP`

In your overlay, add a `patches` section (like in `ligistics_aws_sc/kustomization.yaml`) to replace those names with resources from any folder.
