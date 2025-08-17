---
name: Bug report
about: Create a report to help us improve the Qurtz Helm chart
title: '[BUG] '
labels: 'bug'
assignees: ''

---

## Bug Description
A clear and concise description of what the bug is.

## To Reproduce
Steps to reproduce the behavior:
1. Helm install command used: `helm install qurtz . -f ...`
2. Values configuration used
3. Kubernetes version and cluster details
4. See error

## Expected Behavior
A clear and concise description of what you expected to happen.

## Actual Behavior
What actually happened instead.

## Environment
- **Kubernetes version**: 
- **Helm version**: 
- **Chart version**: 
- **NFS server type**: (e.g., Xpenology, etc.)
- **OS**: 

## Chart Configuration
```yaml
# Please paste your values.yaml configuration here
```

## Logs
```
# Please paste relevant logs here
# kubectl logs deployment/qurtz-watcher -c file-watcher
# kubectl logs deployment/qurtz-watcher -c quartz-builder  
# kubectl logs deployment/qurtz-watcher -c web-server
```

## Additional Context
Add any other context about the problem here, such as:
- Recent changes to the NFS server
- Network configuration
- Other charts running in the same namespace