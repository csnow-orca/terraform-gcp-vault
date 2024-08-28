# Create Snapshot schedule before doing DR configuration

## Create Snapshot schedule
```
vault write sys/storage/raft/snapshot-auto/config/hourly interval="1h" retain=24 file_prefix="PROD" path_prefix="/opt/vault/snapshots/" storage_type="local" local_max_space=10000000
vault list sys/storage/raft/snapshot-auto/config
vault read sys/storage/raft/snapshot-auto/config/hourly
vault delete sys/storage/raft/snapshot-auto/config/hourly
```

## Primary
```
vault write -f sys/replication/dr/primary/enable

vault write sys/replication/dr/primary/secondary-token id=secondary
```

## Secondary
```
vault write sys/replication/dr/secondary/enable token=<TOKEN>

vault read -format=json sys/replication/dr/status
```
## Primary Failover policy
```
vault policy write     dr-secondary-promotion - <<EOF
path "sys/replication/dr/secondary/promote" {
  capabilities = [ "update" ]
}

# To update the primary to connect
path "sys/replication/dr/secondary/update-primary" {
    capabilities = [ "update" ]
}

# Only if using integrated storage (raft) as the storage backend
# To read the current autopilot status
path "sys/storage/raft/autopilot/state" {
    capabilities = [ "update" , "read" ]
}
EOF
```
## Primary Create Batch Token
```
vault write auth/token/roles/failover-handler allowed_policies=dr-secondary-promotion orphan=true renewable=false token_type=batch

vault token create -role=failover-handler -ttl=8h | tee batch.txt
```

## Secondary promote it to Primary
```
vault write sys/replication/dr/secondary/promote dr_operation_token=<DR_TOKEN>
```
## Primary Demote Primary
```
vault write -f sys/replication/dr/primary/demote
```

## Secondary setup replication back to primary
```
vault write sys/replication/dr/primary/secondary-token id=new-secondary
```

## Primary Setup replication back to original primary
```
vault write sys/replication/dr/secondary/update-primary dr_operation_token=<DR_TOKEN> token=<TOKEN>
```

## This command is only if you are not going to have a DR replica.
## Primary Disable replication to make a different cluster DR
```
vault write -f sys/replication/dr/primary/disable
```
