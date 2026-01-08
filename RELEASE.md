## Operator Release Guide
1. Fork https://github.com/redhat-openshift-ecosystem/community-operators-prod.git to ClusterPulse org
2. Run VERSION=0.x.x CHANNELS=fast-v0 DEFAULT_CHANNEL=fast-v0 make bundle bundle-build bundle-push in operator repo 
3. Run cp 0.2.1 0.x.x in forked community-operators-prod
4. Remove everything besides release-config.yaml and copy in the contents of the new bundle
5. commit with -s
6. Run VERSION=0.2.1 make docker-build docker-push
7. Create PR
