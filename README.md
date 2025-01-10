# learning-gitlab-cicd

## Part-1 : Launch a gitlab-server on AWS

- Go to AWS Management Console, we need a `gitlab-server` and we need to install agent into that server.

- Launch an AWS EC2 instance of Amazon Linux 2023 AMI `t2.micro` with security group allowing `SSH:22`, `HTTP:80` and `TCP 8080` ports.

- Make a remote-ssh VSC connection to your instance

```bash
ssh -i .ssh/call-training.pem ec2-user@ec2-3-133-106-98.us-east-2.compute.amazonaws.com
```

- You should write these commands into `.bashrc` file, first go to bottom line and hit enter 2 times then write

```bash (.bashrc)
export PS1="\[\e[1;34m\]\u\[\e[33m\]@\h# \W:\[\e[34m\]\\$\[\e[m\] "  # yellow-blue-yellow
sudo hostnamectl set-hostname "gitlab-server"
```

```bash (pwd : home/ec2-user)
bash

# Update the installed packages and package cache on your instance.
sudo dnf update -y

# install git
sudo dnf install git
git -v      # git version 2.40.1
```

## Part-2 :Create gitlab project/repository `learning-gitlab-cicd` & prepare a runner in gitlab

- Go to gitlab

- Create gitlab project/repository
  Click `+` --> Select `New project/repository` -->> Click `Create blank project`
  `Project name` : `python-app`
  `Project URL` : `https://gitlab.com/arrowlevent/learning-gitlab-cicd`
  `Visibility level` : `Public`
  `Project Configurations` : Put check mark on the `Initialize repository with a README`
  Click `Create project`

- Prepare runner in the gitlab : `new project runner`:`learning-gitlab1` & `Step 1` altındaki komutu `sudo` ekleyerek kopyala
  - Click `Settings` Select `CI/CD` -->> Hit the `Expand` for `Runners`
  - Click `New project runner` ==>> `Platform` - `Operating systems` -> `Linux` (AWS Linux 2023 ec2-instance)
    ==>> `Tags` : `learning-gitlab1` **_pipeline içerisinde her bir job altına bu tag'i kullan ÖNEMLİ, her bir job için ayrı bir tag kullanabilirsin ve böylece her job farklı bir runner içerisinde çalışır_**
    ==>> `Details (optional)` : `*****optional*****`
    `Create runner`
  - `Step 1` altındaki komutu `sudo` ekleyerek terminalde koşmam gerek AMA BU KOMUTTAN ÖNCE gitlab runner'ı agent olarak kurmam gerekiyor.

## Part-3 : Install GitLab Runner manually & Register a runner on gitlab-server ec2-instance

```bash (pwd : home/ec2-user)
# Simply download one of the binaries for your system:
sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64"
# Give it permissions to execute:
sudo chmod +x /usr/local/bin/gitlab-runner
# Create a GitLab CI user:
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
# Install and run as service:
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start

# Register a runner  -->> `Step 1` altındaki `sudo` ekleyerek kopyaladığın komutu yapıştır
sudo gitlab-runner register  --url https://gitlab.com  --token glrt-tQE6qjWfdF2S7QzwdDeT

    Enter the GitLab instance URL (for example, https://gitlab.com/):
    [https://gitlab.com]:   write `https://gitlab.com` then hit enter

    Enter a name for the runner. This is stored only in the local config.toml file:
    [gitlab]:               write `learning` then hit enter        # NE YAZDIĞIN FARKETMEZ

    Enter an executor: docker, virtualbox, docker-autoscaler, kubernetes, custom, docker-windows, parallels, shell, ssh, docker+machine, instance:               write `shell` then hit enter     # job İÇERİSİNDE KOMUTLARIMI shell kullanarak execute edeceğim için

sudo gitlab-runner status
```

## Part-4 : Ensure you have runners available and match with the one in the gitlab-server ec2-instance

- Go to gitlab

  - Click `Settings` Select `CI/CD` -->> Hit the `Expand` for `Runners` ==>> See `Assigned project runners` under `Project runners` -- Go to green circle and see the note `Runner is online; last connect was 10 minutes ago` ==>> Burada gördüğün `#30870596 (tQE6qjWfd)` ifadesi benim ec2-instance içindeki ile eşleşmesi lazım

- Go to AWS Management Console - gitlab-server

```bash (pwd : home/ec2-user)
sudo gitlab-runner verify
        Verifying runner... is valid                        runner=sJBzBRhDm        # yukarıdaki ile aynı
```

***Pipeline hazırlayıp Click `Commit changes` yapmadan önce, pipeline içinde kullanacağımız `predefined variable` OLMAYAN variable varsa, sensitive veya sensitive olmayan variables tanımlaması yapalım. Sensitive olmayanları direkt olarak pipeline içine de yazabilirim.***

**_Örnekte bu kısmı kullanmıyorum, direkt olarak pipeline içerisinde sensitive olmayan variable kullanıyorum, sadece gitlab dashboard üzerinden sensitive ve sensitive olmayan variable kullanımına örnek olması açısından yazdım_**

- Define a CI/CD variable in the UI: `Sensitive variables` like tokens or passwords should be stored in the settings in the UI, not in the .gitlab-ci.yml file. Define CI/CD variables in the UI:
  For a project in the project’s settings. [https://docs.gitlab.com/ee/ci/variables/index.html#for-a-project]

- Pipeline hazırlamadan önce pipeline içinde kullanacağımız `sensitive` veya `sensitive olmayan` variables tanımlaması :
  Sensitive olan için ayrıca `Mask variable` EKLİYORUZ.
   - `Settings` --> `CI/CD` --> `Variables` - `Expand` --> `Add variable` ==> `Key` : `DOCKER_HUB_USER`
                                                                          ==> `Value` : `alparslanu6347`  ***write yours*** 
                                                                                                          --> Click `Add variable`
                                                                          ==> `Type` : `Default`
                                                                          ==> `Environment` : `Default`
                                                       --> `Add variable` ==> `Key` : `DOCKER_HUB_PWD`
                                                                          ==> `Value` : `************`  ***write yours***
                                                                          ==> `Type` : `Default`
                                                                          ==> `Environment` : `Default`
                                                                          İLAVE OLARAK put check mark `Mask variable` --> Click `Add variable`

## Part-5 : Prepare `.gitlab-ci.yml` and deploy

- Bu pipeline içerisinde en başta stage bölümünde stage isimlerini listeleyebiliriz, ama yazmasak da çalışır.

- Go to gitlab
  - Click and get into your project/repository `learning-gitlab-cicd` --> Click `+` Select ``New file` from the list
    `filename` : `.gitlab-ci.yml`
    copy-paste into this file

```yaml (.gitlab-ci.yml)
variables:
  GLOBAL_VAR: "A global variable"

job-var:
  variables: {}
  script:
    - echo This job does not need any variables

build-job:
  variables:
    JOB_VAR: "A job variable"
  stage: build
  script:
    - echo "Hello, $GITLAB_USER_LOGIN!"
    - echo "The job's stage is '$CI_JOB_STAGE'"
    - echo "Variables are '$GLOBAL_VAR' and '$JOB_VAR'"

test-job1:
  stage: test
  script:
    - echo "This job tests something"
    - echo "The job's stage is '$CI_JOB_STAGE'"
    - echo "Variables are '$GLOBAL_VAR' and '$JOB_VAR'"

test-job2:
  stage: test
  script:
    - echo "This job tests something, but takes more time than test-job1."
    - echo "After the echo commands complete, it runs the sleep command for 20 seconds"
    - echo "which simulates a test that runs 20 seconds longer than test-job1"
    - sleep 20

deploy-prod:
  stage: deploy
  script:
    - echo "The job's stage is '$CI_JOB_STAGE'"
    - echo "This job deploys something from the $CI_COMMIT_BRANCH branch."
  environment: production
```

- Click `Commit changes`
- Pipeline aşamalarını/joblarını buradan gözlemleyebilirsin`Build` -- `Pipelines`

- alternatif pipeline

```yaml (.gitlab-ci.yml)
stages:
  - build
  - test
  - deploy

variables:
  GLOBAL_VAR: "A global variable"

build-job:
  variables:
    JOB_VAR: "A job variable"
  stage: build
  script:
    - echo "Hello, $GITLAB_USER_LOGIN!"
    - echo "The job's stage is '$CI_JOB_STAGE'"
    - echo "Variables are '$GLOBAL_VAR' and '$JOB_VAR'"

test-job1:
  stage: test
  script:
    - echo "This job tests something"
    - echo "The job's stage is '$CI_JOB_STAGE'"
    - echo "Variables are '$GLOBAL_VAR' and '$JOB_VAR'"

test-job2:
  stage: test
  script:
    - echo "This job tests something, but takes more time than test-job1."
    - echo "After the echo commands complete, it runs the sleep command for 20 seconds"
    - echo "which simulates a test that runs 20 seconds longer than test-job1"
    - sleep 20

deploy-prod:
  stage: deploy
  script:
    - echo "The job's stage is '$CI_JOB_STAGE'"
    - echo "This job deploys something from the $CI_COMMIT_BRANCH branch."
  environment: production
```
- Click `Commit changes`

- Pipeline aşamalarını/joblarını buradan gözlemleyebilirsin`Build` -- `Pipelines`

## Resources

- https://gitlab.com/arrowlevent/learning-gitlab-cicd