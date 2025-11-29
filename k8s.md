# K8S record and replay for debugging faults  

Recording only the control plane is easy and a research project [simkube](https://github.com/acrlabs/simkube) is doing this.

However, in production environments, faults can surface at any level(Application, virtualization, kenrl and hardware) of the system. 

Treating a pod as a blackbox ?


