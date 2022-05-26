# Deploy MicroShift on a OSTree based system
MicroShift is a research project which aims to reduce Kubernetes and OKD (the Kubernetes distribution by the OpenShift community) footprints, in order to be deployed on edge computing devices. According to security standards, an edge device deployed in the field should run an operating system optimizd to be inmutable, and based in transactions for rollbacks and upgrades. Those transactions should be done in a securely and safely way, in addition to be speedly, but always avoiding workloads disrruptions. All those requirements metioned before could be addressed by running MicroShift in a OSTree based system. 

## OSTree based systems


