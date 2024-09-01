The folder kube-manifests contain a collection of yaml files designed to deploy the tasky application to an existing kubernetes cluster.

It deploys a single pod running tasky on Amazon EKS.  A service of type loadBalancer provides external access to the web application.

Tasky uses a mongoDB back-end.  For information on Tasky, refer to https://github.com/jeffthorne/tasky
