
# `minio-microk8s-ansible` – MinIO S3 Object Storage with MicroK8s Load Balancing/Ingress

Ansible playbook to configure our Ubuntu 22 servers to run a distributed MinIO S3 service. MicroK8s
is used here as a sidecar to provide container platform capabilities, load balancing, and handle
internet ingress.

## Getting Started

Clone the repo:

    git clone git@github.com:OwnTube-tv/minio-microk8s-ansible.git
    cd minio-microk8s-ansible/

Create a virtual environment and install the dependencies:

    python3 -m venv venv
    source venv/bin/activate
    pip install -r requirements.txt

Add the Ansible Vault password to a file named `.ansible_vault_password` and restrict readability:

    echo theSecretAnsibleVaultPassword > .ansible_vault_password
    chmod og-r .ansible_vault_password

Verify that the hosts are reachable:

    ansible minio_microk8s_servers -m ping

Run through the bootstrap playbook in `--check` mode to verify that provisioning can execute:

    ansible-playbook 0-bootstrap.yml --check


## Live Deployment

### Initial Setup

The initial setup steps for a live deployment are as follows:

1. Run the `0-bootstrap.yml` playbook to prepare the server baseline for MinIO and MicroK8s setup:

    ```shell
    ansible-playbook 0-bootstrap.yml
    ```

    Follow the instructions in the end of the playbook to establish HA clustering for MicroK8s.

2. Run the `1-microk8s-cluster.yml` playbook to set up MicroK8s add-ons and configure the cluster:

    ```shell
    ansible-playbook 1-microk8s-cluster.yml
    ```

    After the successful completion of the playbook, you can access the Kubernetes dashboard at
    https://k8s-dashboard.owntube.tv/ with a proper certificate and login with a token created from
    one of the MicroK8s cluster nodes:

    ```shell
    kubectl get secret -n kube-system microk8s-dashboard-token \
      -o jsonpath="{.data.token}" | base64 -d
    ```

3. Run the `2-minio-servers.yml` playbook to set up the MinIO S3 object storage service:

    ```shell
    ansible-playbook -e @secrets.yml 2-minio-servers.yml
    ```

    After the successful completion of the playbook, you can access the MinIO web interface at
    https://minio.owntube.tv/ and be able to log in with the root username and password.


### Add OpenID Connect using Auth0

Setup steps to integrate MinIO with Auth0 OpenID Connect for user authentication and authorization:

1.  Create an OpenID Connect application in your Auth0 tenant the following parameters:

    ```properties
    application_type=Regular Web Application
    login_url=https://minio.owntube.tv/console/
    callback_urls=https://minio.owntube.tv/console/oauth_callback
    logout_urls=https://minio.owntube.tv/console/
    allowed_web_origins=https://minio.owntube.tv
    ```

2.  Configure your Auth0 tenant with the Auth0 _PostLogin_ Action
    ["Add MinIO Policy OpenID Claim"](https://github.com/auth0/opensource-marketplace/blob/main/templates/add-minio-policy-open-id-claim-POST_LOGIN)
    and set the following "secrets":

    ```properties
    POST_LOGIN_MINIO_CLAIM_PREFIX=https://minio.owntube.tv/console/
    POST_LOGIN_MINIO_CLAIM_DEFAULT_POLICY=noaccess
    POST_LOGIN_MINIO_CLAIM_USER_POLICY_MAP={"mats.blomdahl@gmail.com":"consoleAdmin,diagnostics","sasha@mkdevops.se":"swt-readwrite,ot-readwrite","bot@mkdevops.se":"ot-readwrite","viktor.v.karlsson@hotmail.com":"ot-readwrite","bwende-d@hotmail.com":"ot-readwrite"}
    ```

3.  Configure the Ansible project `secrets.yml` with the config URL, client ID and client secret for
    the OpenID application (from setup step 1):

    ```yaml
    minio_auth0_oauth_config_url: https://owntube-tv.eu.auth0.com/.well-known/openid-configuration
    minio_auth0_oauth_client_id: XIa**************************MzK
    minio_auth0_oauth_client_secret: 6-B**********************************************************koW
    ```

4.  Run the `3-minio-auth0-oidc.yml` playbook to configure MinIO with Auth0 OpenID Connect:

    ```shell
    ansible-playbook -e @secrets.yml 3-minio-oidc.yml
    ```

    After the successful completion of the playbook, you can access the MinIO web interface at
    https://minio.owntube.tv/ and find that the old username/password form have been replaced by a
    button with the text _"GitHub-Auth0 authentication"_ and be able to authenticate using your
    GitHub identity. When returning to the login screen after GitHub-Auth0 authentication, you will
    find an error about the JWT Claim for policy does not exist; continue and set up in step 5.

5.  Login to the MinIO web interface as admin using the _"Other Authentication Methods" >
    "Use Credentials"_ drop-down menu and create the following policies and buckets:

    1. Create a bucket named `"auth0-openid-noaccess"`, then configure a policy named `"noaccess"`
       for unknown users to login and have only read access to the `"auth0-openid-noaccess"` bucket:

        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "s3:GetBucketLocation",
                        "s3:GetObject",
                        "s3:List*",
                        "s3:ListBucket"
                    ],
                    "Resource": [
                        "arn:aws:s3:::auth0-openid-noaccess*"
                    ]
                }
            ]
        }
        ```

    2.  Create a bucket named `"swt-pt-dev-1"`, then configure a policy named `"swt-readwrite"`
        for special users that are mapped to this role using the Auth0 Action (setup step 2):

        ```json
        {
              "Version": "2012-10-17",
              "Statement": [
                  {
                      "Effect": "Allow",
                      "Action": [
                          "s3:*"
                      ],
                      "Resource": [
                          "arn:aws:s3:::swt-*"
                      ]
                  }
              ]
          }
          ```

    3.  Create a bucket named `"ot-pt-dev-1"`, then configure a policy named `"ot-readwrite"` for
        special users that are mapped to this role using the Auth0 Action (setup step 2):

        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "s3:*"
                    ],
                    "Resource": [
                        "arn:aws:s3:::ot-*"
                    ]
                }
            ]
        }
        ```

6.  Verify that the Auth0 OpenID Connect integration works by logging in to
    https://minio.owntube.tv/ with ...

    1.  a user that does not have a policy mapped to it, expect to only see the
        `"auth0-openid-noaccess"` bucket listed in the _Object Browser_, with read-only access only

    2. a user that has its email mapped to the policy `"ot-readwrite"`, expect to only see the
        `"ot-pt-dev-1"` bucket listed in the _Object Browser_ and verify that the user is able to
        administer the bucket via https://minio.owntube.tv/console/buckets/ot-pt-dev-1/admin/

    3.  a user that has its email mapped to the policy `"swt-readwrite"`, expect to only see the
        `"swt-pt-dev-1"` bucket listed in the _Object Browser_ and verify that the user is able to
        administer the bucket via https://minio.owntube.tv/console/buckets/swt-pt-dev-1/admin/


## Contact

For ideas on enhancements, discussing worthwhile feature to have, or if you wish to contribute
improvements on your own, please reach out to `@mblomdahl` :zap: by opening
[a new _Issue_](https://github.com/OwnTube-tv/minio-microk8s-ansible/issues/new).
