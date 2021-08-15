# MOSIP Sandbox-v2 v1.1.2 Installation Guide for AWS 
This guide is based on instructions from the MOSIP github repository. When installed, it should consist of thirteen virtual machines of instance type `m5a.xlarge`. These are viewable from the EC2 console in the ap-south-1(Mumbai) region.
## Part 1: VM setup 
Based on instructions at the MOSIP Github page [https://github.com/mosip/mosip-infra/tree/1.1.2/deployment/sandbox-v2/terraform/aws/sandbox] for the AWS Sandbox
1. Clone repo from link above, use branch 1.1.2 
2. Install terraform [https://learn.hashicorp.com/tutorials/terraform/install-cli]
3. Set up environment variables,using following commands:
    ```
    export AWS_ACCESS_KEY_ID=<> 
    export AWS_SECRET_ACCESS_KEY=<> 
    export TF_LOG=DEBUG 
    export TF_LOG_PATH=tf.log 
    ```
To get an AWS_ACCESS_KEY follow instructions here [https://docs.aws.amazon.com/powershell/latest/userguide/pstools-appendix-sign-up.html]

4. On the AWS EC2 admin console generate a key pair called  `mosip-aws`. Download the private key `mosip-aws.pem` to your local `~/.ssh` folder. Make sure the permission of `~/.ssh/mosip-aws.pem` is set to `600` using `chmod`. If you wish to name the key something else, alter the key name in the `variables.tf` file 
5. Generate a new set of RSA keys with default names `id_rsa` and `id_rsa.pub` and put them in the directory with the terraform scripts using following command: 
    `ssh-keygen -t rsa -f ./id_rsa `
6. Run terraform: 
   ```
    terraform init # One time 
    terraform plan 
    terraform apply
   ``` 
The setup requires at least 52 vCPUs (4 each for 13 machines). If this amount is above your limit submit a ticket for a limit increase of on-demand instances.

## Part 2: Installation setup of MOSIP 
Based on instructions at MOSIP Github repository [https://github.com/mosip/mosip-infra/tree/1.1.2/deployment/sandbox-v2]
1. Once initialized, log into console.sb using command 
`ssh -i ~/.ssh/mosip-aws.pem centos@<PUBLIC_IP>` Public IP is shown in as public IPv4 on EC2 Dashboard 
2. Change to mosipuser: 
    `sudo su mosipuser` 
3. Install git: 
    `sudo yum install -y git` 
4. Git clone this repo in home directory: 
    ```
    cd ~/ 
    git clone https://github.com/mosip/mosip-infra $ cd mosip-infra 
    git checkout 1.1.2 
    cd mosip-infra/deployment/sandbox-v2
    ``` 
5. Make mosipuser owner of the cloned repo: 
    `sudo chown -R mosipuser ~/mosip-infra/`
6: Install Ansible and create shortcuts: 
    ```
    ./preinstall.sh 
    source ~/.bashrc
    ```
## Part 3: Replacement and editing of files 
Updates several files that cause issues during installation. Sources for any borrowed solution will be linked. The order that these edits are implemented in doesn’t matter. 

1. In `roles/packages/helm-cli/tasks/main.yml` replace 
https://kubernetes-charts.storage.googleapis.com with 
https://charts.helm.sh/stable (source: https://stackoverflow.com/questions/61954440/how-to-resolve-https-kubernetes-charts-storage-googleapis-com-is-not-a-valid/65404574#65404574)
2. In `roles/k8scluster/kubernetes/node/meta/main.yml` and 
`roles/k8scluster/kubernetes/master/meta/main.yml`, replace package names with `packagename-1.19.0`. For instance, `kubeadm` becomes `kubeadm-1.19.0`. (source: https://github.com/luker983/MOSIP-Setup-Instructions/tree/1.1.2) 
3. In `roles/packages/psql/tasks/main.yml` replace 
https://download.postgresql.org/pub/repos/yum/12/redhat/rhel-7-x86_64/pgdg-re dhat-repo-latest.noarch.rpm with 
https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redha t-repo-latest.noarch.rpm 
4. In `roles/packages/psycopg2/tasks/main.yml` specify version `2.8.6` for psycopg2-binary. So replace both instances of `psycopg2-binary` with `psycopg2-binary==2.8.6`. 
5. If you are using a selfsigned certificate: open 
`mosip-infra/deployment/sandbox-v2/group_vars/all.yml` and replace following values as shown below: 
```
    sandbox_domain_name: '{{inventory_hostname}}' 
    site: 
    sandbox_public_url: 'https://{{sandbox_domain_name}}' 
    ssl: 
    ca: 'selfsigned' # The ca to be used in this deployment
``` 
6. Install nginx module correctly: 
    `sudo yum install nginx-mod-stream` (source: https://serverfault.com/questions/858067/unknown-directive-stream-in-etc-nginx-nginx-conf86)

## Part 4: Installation and Installation Issues 
Run installation script from sandbox-v2 directory. If installation encounters any known errors, fix the issue and rerun the script 
1. Run ansible scripts that install MOSIP: 
   ```
    sb 
    an site.yml 
   ```
    
## Issues 
### Error 1: 
#### Output:
```
TASK [packages/crypto : Install python3 cryptography] 
*************************************************************************************** fatal: [console.sb]: FAILED! => {"changed": false, "cmd": ["/bin/pip3", "install", "cryptography"], "msg": "stdout: Collecting cryptography\n Downloading 
https://files.pythonhosted.org/packages/9b/77/461087a514d2e8ece1c975d8216bc03f7048e6 090c5166bc34115afdaa53/cryptography-3.4.7.tar.gz (546kB)\n Complete output from command python setup.py egg_info:\n \n 
=============================DEBUG 
ASSISTANCE==========================\n If you are seeing an error here please try the following to\n successfully install cryptography:\n \n Upgrade to the latest pip and try again. This will fix errors for most\n users. See: https://pip.pypa.io/en/stable/installing/#upgrading-pip\n 
=============================DEBUG 
ASSISTANCE==========================\n \n Traceback (most recent call last):\n File \"<string>\", line 1, in <module>\n File 
\"/tmp/pip-build-u9o62e78/cryptography/setup.py\", line 14, in <module>\n from setuptools_rust import RustExtension\n ModuleNotFoundError: No module named 'setuptools_rust'\n \n ----------------------------------------\n\n:stderr: WARNING: Running pip install with root privileges is generally not a good idea. Try `pip3 install --user` instead.\nCommand \"python setup.py egg_info\" failed with error code 1 in /tmp/pip-build-u9o62e78/cryptography/\n"} 
```

#### Fix: 
First, update pip3 and rerun script:
    ```
    sudo pip3 install --upgrade pip 
    an site.yml 
    ```

Another error should appear after the initial one: 
```
TASK [ssl/letsencrypt : Install certbot in python3 virtual environment] 
********************************************************************* 
fatal: [console.sb]: FAILED! => {"changed": false, "cmd": 
["/home/mosipuser/.venv-py3/bin/pip", "install", "certbot==1.3.0"], "msg": "stdout: Using base prefix '/usr'\nNew python executable in /home/mosipuser/.venv-py3/bin/python3\nAlso creating executable in /home/mosipuser/.venv-py3/bin/python\nInstalling setuptools, pip, wheel...done.\nRunning virtualenv with interpreter /usr/bin/python3\nCollecting certbot==1.3.0\n Downloading 
https://files.pythonhosted.org/packages/4b/3d/afa627553cdd9b69553637fd15d07bee32f31e9 401e5413fd7806367e54a/certbot-1.3.0-py2.py3-none-any.whl (231kB)\nCollecting josepy>=1.1.0 (from certbot==1.3.0)\n Downloading 
https://files.pythonhosted.org/packages/6d/18/970ee71ffe3a45e5bddcf45f32efb60d7863bfdbd a3083742144f40f8410/josepy-1.8.0-py2.py3-none-any.whl (61kB)\nCollecting mock (from certbot==1.3.0)\n Downloading 
https://files.pythonhosted.org/packages/5c/03/b7e605db4a57c0f6fba744b11ef3ddf4ddebcada 35022927a2b5fc623fdf/mock-4.0.3-py3-none-any.whl\nCollecting acme>=0.40.0 (from certbot==1.3.0)\n Downloading 
https://files.pythonhosted.org/packages/21/51/cb5116dec68fbaf31ae065fff2c714dc414f5b61df f45716c3bb4de20efe/acme-1.16.0-py2.py3-none-any.whl (42kB)\nCollecting zope.component (from certbot==1.3.0)\n Downloading 
https://files.pythonhosted.org/packages/0d/df/e9f3cf96309dd84f24ad842eb41969bb5085bafb 92efb19443f9ddf9beb2/zope.component-5.0.0-py2.py3-none-any.whl (68kB)\nCollecting pyrfc3339 (from certbot==1.3.0)\n Downloading 
https://files.pythonhosted.org/packages/c1/7a/725f5c16756ec6211b1e7eeac09f46908459551 3917ea069bc023c40a5e2/pyRFC3339-1.1-py2.py3-none-any.whl\nCollecting zope.interface (from certbot==1.3.0)\n Downloading 
https://files.pythonhosted.org/packages/67/67/8178e511cd4f0a481082aac1c0e2d64c520a5ee 92ea8ce42d8297a4fca7e/zope.interface-5.4.0-cp36-cp36m-manylinux1_x86_64.whl (251kB)\nCollecting cryptography>=1.2.3 (from certbot==1.3.0)\n Downloading https://files.pythonhosted.org/packages/9b/77/461087a514d2e8ece1c975d8216bc03f7048e6 090c5166bc34115afdaa53/cryptography-3.4.7.tar.gz (546kB)\n Complete output from command python setup.py egg_info:\n \n 
=============================DEBUG 
ASSISTANCE==========================\n If you are seeing an error here please try the following to\n successfully install cryptography:\n \n Upgrade to the latest pip and try again. This will fix errors for most\n users. See: https://pip.pypa.io/en/stable/installing/#upgrading-pip\n
=============================DEBUG 
ASSISTANCE==========================\n \n Traceback (most recent call last):\n File \"<string>\", line 1, in <module>\n File 
\"/tmp/pip-build-jslommyk/cryptography/setup.py\", line 14, in <module>\n from setuptools_rust import RustExtension\n ModuleNotFoundError: No module named 'setuptools_rust'\n \n ----------------------------------------\n\n:stderr: Command \"python setup.py egg_info\" failed with error code 1 in /tmp/pip-build-jslommyk/cryptography/\nYou are using pip version 9.0.1, however version 21.1.3 is available.\nYou should consider upgrading via the 'pip install --upgrade pip' command.\n"} 
```

Follow the below steps to fix the above error:
```
    source /home/mosipuser/.venv-py3/bin/activate 
    python3 -m pip install setuptools_rust 
    pip install --upgrade pip 
    python -m pip install certbot 
    deactivate 
```

The reason these two errors are grouped together is because the virtual environments aren’t installed until after the first issue is fixed which only requires a pip3 update.