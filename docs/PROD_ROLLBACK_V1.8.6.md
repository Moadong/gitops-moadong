# Production rollback: v1.8.6

## Summary

On 2026-10-06, the production deployment of `moadong/moadong-release:v1.8.6`
failed to start. This GitOps change rolls production back to the last healthy
image, `v1.8.5`.

## Impact

The `v1.8.6` ReplicaSet entered `CrashLoopBackOff`, leaving three `v1.8.5`
pods serving production traffic. The rollout exceeded its progress deadline,
so the Argo CD application reported `Degraded`.

## Root cause

During Spring Data MongoDB initialization, `v1.8.6` attempted to create the
`student_users.studentId` index as `unique=true, sparse=true`. Production
already has an index named `studentId` on the same key with
`unique=true, sparse=false`. MongoDB rejected the conflicting index
specification with `IndexKeySpecsConflict` (error 86), preventing application
startup.

## Re-deploy criteria for v1.8.6 or a successor

Before updating the production image tag again:

1. Make the application's index declaration match the existing production
   index, or execute a reviewed, reversible MongoDB index migration.
2. Validate application startup against an environment containing the migrated
   index.
3. Confirm the new ReplicaSet becomes Ready before scaling down the prior
   healthy ReplicaSet.

## Verification after this rollback

Argo CD must report `moadong-backend-prod` as `Synced` and `Healthy`, and the
`moadong` Deployment must report four available replicas with no
`CrashLoopBackOff` pods.
