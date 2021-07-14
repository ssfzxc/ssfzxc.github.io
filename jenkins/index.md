# Hello Jenkins



# Hello Jenkins


> 安装 Jenkins 时的一些笔记



## 坑

- Jenkins Job 启动后服务自动随 Job 执行完毕后关闭

  - Pipeline：新增 withEnv(['JENKINS_NODE_COOKIE=background_job'])

    ```groovy
    pipeline {
        agent any

        stages {
            stage('startup service local') {
                steps {
                    script {
                        withEnv(['JENKINS_NODE_COOKIE=background_job']) {
                            sh 'echo hello'
                        }
                    }
                }
            }
        }
    }
    ```

  - FreeStyle: 脚本最前面新增 BUILD_ID

  ```sh
  BUILD_ID=donotKillMe
  echo 'hello'
  ```


