---
title: devops之KubeSphere流水线部署
date: 2022-07-26 12:38:02
categories:
  - 服务器
  - k8s
tags:
  - kubernetes 
  - k8s
  - kubesphere
  - 流水线
---

![流水线概览](devops之KubeSphere流水线部署/pipeline-overview.png) 

- **阶段 1：Checkout SCM**：从 GitHub 仓库检出源代码。
- **阶段 2：单元测试**：待该测试通过后才会进行下一阶段。
- **阶段 3：SonarQube 分析**：SonarQube 代码质量分析。
- **阶段 4：构建并推送快照镜像**：根据**策略设置**中选定的分支来构建镜像，并将 `SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER` 标签推送至 Docker Hub，其中 `$BUILD_NUMBER` 为流水线活动列表中的运行序号。
- **阶段 5：推送最新镜像**：将 SonarQube 分支标记为 `latest`，并推送至 Docker Hub。
- **阶段 6：部署至开发环境**：将 SonarQube 分支部署到开发环境，此阶段需要审核。
- **阶段 7：带标签推送**：生成标签并发布到 GitHub，该标签会推送到 Docker Hub。
- **阶段 8：部署至生产环境**：将已发布的标签部署到生产环境。

# 一、开启devops

cluster-configuration.yaml

```
devops:
  enabled: true # 将“false”更改为“true”。
```

执行：

```
kubectl apply -f cluster-configuration.yaml
```

查看安装情况：

```
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
```

查看结果：

```
[root@k8s-master01 kubesphere]# kubectl get pod --all-namespaces
NAMESPACE                      NAME                                                              READY   STATUS      RESTARTS         AGE
argocd                         devops-argocd-application-controller-0                            0/1     Running     9 (29s ago)      24m
argocd                         devops-argocd-applicationset-controller-b88d4b875-hztm8           1/1     Running     0                24m
argocd                         devops-argocd-dex-server-5f4c69cdb8-26d9x                         1/1     Running     0                24m
argocd                         devops-argocd-notifications-controller-6d86f8974f-twlk6           1/1     Running     0                24m
argocd                         devops-argocd-redis-655969589d-vbzfq                              1/1     Running     0                24m
argocd                         devops-argocd-repo-server-f77687668-l46fj                         1/1     Running     0                24m
argocd                         devops-argocd-server-6c55bbb84f-tgdv6                             1/1     Running     0                24m
....
```

![1662990207401](devops之KubeSphere流水线部署/1662990207401.png)

jdk11:

在**配置字典**页面的搜索框中输入 `jenkins-casc-config` 并按**回车键**。 



# 二、创建凭证

## 1.1 创建dockerhub凭证

1. 登录 Docker Hub，点击右上角的帐户，并从菜单中选择 **Account Settings**。

2. 在左侧导航栏点击 **Security**，然后点击 **New Access Token**。

3. 在弹出的对话框中，输入令牌名称（`go-project-token`），点击 **Create**。

   ```
   dockerhub-id
   用户名：leellun
   密码：dckr_pat_P9abXAwRM9TU96yyERoODoyThRY
   ```

4. 点击 **Copy and Close** 并务必保存该访问令牌。

![1662995312671](devops之KubeSphere流水线部署/1662995312671.png)

## 1.2 github凭证

github-id

用户名和token添加

## 1.3 kubeconfig凭证添加

可以用root/.kube/config内容，也可以生成用户认证kubeconfig

![1662995264415](devops之KubeSphere流水线部署/1662995264415.png)

## 1.4 SonarQube凭证添加

sonar-token

进入sonarqube添加项目并且生成token：**caea718fe1c8d61628c85cc642998605247c4de9** 

![1662997272162](devops之KubeSphere流水线部署/1662997272162.png) 

配置sonarqube凭证：

用户名和token添加

```
user: 
token: caea718fe1c8d61628c85cc642998605247c4de9
```



# 三、修改 Jenkinsfile

![jenkins-edit--2](devops之KubeSphere流水线部署/jenkins-edit--2.png)

| 条目                     | 值                     | 描述信息                                                     |
| ------------------------ | ---------------------- | ------------------------------------------------------------ |
| DOCKER_CREDENTIAL_ID     | dockerhub-id           | 您在 KubeSphere 中为 Docker Hub 帐户设置的**名称**。         |
| GITHUB_CREDENTIAL_ID     | github-id              | 您在 KubeSphere 中为 GitHub 帐户设置的**名称**，用于将标签推送至您的 GitHub 仓库。 |
| KUBECONFIG_CREDENTIAL_ID | demo-kubeconfig        | 您在 KubeSphere 中为 kubeconfig 设置的**名称**，用于访问运行中的 Kubernetes 集群。 |
| REGISTRY                 | docker.io              | 默认为 `docker.io`，用作推送镜像的地址。                     |
| DOCKERHUB_NAMESPACE      | your-dockerhub-account | 请替换为您的 Docker Hub 帐户名，也可以替换为该帐户下的 Organization 名称。 |
| GITHUB_ACCOUNT           | your-github-account    | 请替换为您的 GitHub 帐户名。例如，如果您的 GitHub 地址是 `https://github.com/kubesphere/`，则您的 GitHub 帐户名为 `kubesphere`，也可以替换为该帐户下的 Organization 名称。 |
| APP_NAME                 | devops-maven-sample    | 应用名称。                                                   |
| SONAR_CREDENTIAL_ID      | sonar-token            | 您在 KubeSphere 中为 SonarQube 令牌设置的**名称**，用于代码质量检测。 |

# 四、具体Jenkinsfile根据项目而定

下面选用的代码仓库是码云、docker镜像仓库是dockerhub、代码分析sonar是自己部署的

流水线步骤：拉取镜像——单元测试——代码质量分析——编译 & 推送——推送最新——发布开发环境——推送正式版本——发布生产环境

```
pipeline {
  agent {
    node {
      label 'maven'
    }

  }
  environment {
    DOCKER_CREDENTIAL_ID = 'dockerhub-id'
    GITEE_CREDENTIAL_ID = 'gitee-id'
    KUBECONFIG_CREDENTIAL_ID = 'kubeconfig'
    REGISTRY = 'docker.io'
    DOCKERHUB_NAMESPACE = 'leellun'
    GIT_ACCOUNT = 'myselfyou'
    SONAR_CREDENTIAL_ID = 'sonar-token'
    BRANCH_NAME = 'master'
  }
  parameters {
    string(name: 'PROJECT_VERSION', defaultValue: '', description: '版本')
    string(name: 'PROJECT_NAME', defaultValue: '', description: '项目名称')
  }
  stages {
    stage('拉取代码') {
      steps {
        sh "echo 正在构建 $PROJECT_NAME 版本号：$PROJECT_VERSION 将会提交给 $REGISTRY 镜像仓库"
        git(url: 'https://gitee.com/myselfyou/gulimall-learng.git', credentialsId: "$GITEE_CREDENTIAL_ID", branch: 'master', changelog: true, poll: false)
      }
    }
    stage ('单元测试') {
        steps {
            container ('maven') {
                sh 'java -version'
                sh 'mvn clean test'
            }
        }
    }
    stage('代码质量分析') {
      steps {
        container('maven') {
          withCredentials([string(credentialsId: "$SONAR_CREDENTIAL_ID", variable: 'SONAR_TOKEN')]) {
            withSonarQubeEnv('sonar') {
              sh "mvn sonar:sonar -Dsonar.login=$SONAR_TOKEN"
            }
          }
        }
      }
    }
    stage ('编译 & 推送') {
        steps {
            container ('maven') {
                sh 'mvn clean package -DskipTests'
                sh 'cd $PROJECT_NAME && docker build -f Dockerfile -t $REGISTRY/$DOCKERHUB_NAMESPACE/$PROJECT_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER .'
                withCredentials([usernamePassword(passwordVariable : 'DOCKER_PASSWORD' ,usernameVariable : 'DOCKER_USERNAME' ,credentialsId : "$DOCKER_CREDENTIAL_ID" ,)]) {
                    sh 'echo "$DOCKER_PASSWORD" | docker login $REGISTRY -u "$DOCKER_USERNAME" --password-stdin'
                    sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/$PROJECT_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER'
                }
            }
        }
    }
    stage('推送最新'){
       when{
         branch 'master'
       }
       steps{
            container ('maven') {
              sh 'docker tag  $REGISTRY/$DOCKERHUB_NAMESPACE/$PROJECT_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$PROJECT_NAME:latest '
              sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/$PROJECT_NAME:latest '
            }
       }
    }
    stage('发布开发环境') {
      when{
        branch 'master'
      }
      steps {
        input(id: 'deploy-to-dev', message: '是否发布到开发环境?')
        container ('maven') {
            withCredentials([
                kubeconfigFile(
                credentialsId: env.KUBECONFIG_CREDENTIAL_ID,
                variable: 'KUBECONFIG')
                ]) {
                sh 'envsubst < $PROJECT_NAME/deploy/$PROJECT_NAME-deploy.yaml | kubectl apply -f -'
            }
        }
      }
    }
    stage('推送正式版本'){
      when{
        expression{
          return params.PROJECT_VERSION =~ /v.*/
        }
      }
      steps {
          container ('maven') {
            input(id: 'release-image-with-tag', message: '是否发布镜像版本$PROJECT_NAME:$PROJECT_VERSION?')
              withCredentials([usernamePassword(credentialsId: "$GITEE_CREDENTIAL_ID", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                sh 'git config --global user.email "kubesphere@qq.com" '
                sh 'git config --global user.name "kubesphere" '
                sh 'git tag -a $PROJECT_NAME-$PROJECT_VERSION -m "$PROJECT_NAME-$PROJECT_VERSION" '
                sh 'echo $GIT_ACCOUNT'
                sh 'git push http://$GIT_USERNAME:$GIT_PASSWORD@gitee.com/$GIT_ACCOUNT/gulimall-learng.git --tags --ipv4'
              }
            sh 'docker tag  $REGISTRY/$DOCKERHUB_NAMESPACE/$PROJECT_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$PROJECT_NAME:$PROJECT_VERSION '
            sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/$PROJECT_NAME:$PROJECT_VERSION '
        }
      }
    }
    stage('发布生产环境') {
      when{
        expression{
          return params.PROJECT_VERSION =~ /v.*/
        }
      }
      steps {
        input(id: 'deploy-to-production', message: '是否发布生产环境?')
        container ('maven') {
            withCredentials([
                kubeconfigFile(
                credentialsId: env.KUBECONFIG_CREDENTIAL_ID,
                variable: 'KUBECONFIG')
                ]) {
                sh 'envsubst < $PROJECT_NAME/deploy/$PROJECT_NAME-deploy.yaml | kubectl apply -f -'
            }
        }
      }
    }
  }
}
```

