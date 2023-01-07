#### Kubernetes Security Primitives
- important to control access to the kube-apiserver
- Authorization is implemented with role-based action control 
- TLS encryption secures all communication between kube-apiserver and other components
___
#### Authentication
- Kubernetes does manage user accounts
- depends on file with user details, certs, or LDAP
- service accounts can be managed
kube-apiserver can auth with static token/password file, certs, or identity services (LDAP, Kerberos)
- `--basic-auth-file=user-details.csv` in pod definition file or kube-apiserver.service
- 