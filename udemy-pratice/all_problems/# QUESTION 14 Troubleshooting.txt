# QUESTION 14: Troubleshooting

After a cluster migration, the controlplane kube-apiserver is not coming up.

**Before migration:**
- etcd was external and in HA

**After migration:**
- kube-apiserver was pointing to etcd peer port 2380 instead of 2379

**Task:**
Fix it.
