***Jenkins Pipeline to Trigger Automatically When a New User is Added***

 **Configure GitHub Webhook for Auto-Trigger**
1. Go to GitHub Repository → Settings → Webhooks
2. Click "Add Webhook"
2. Set the Payload URL as:
```
http://your-jenkins-server/github-webhook/
```

4. Content type: application/json
5. Trigger Event: Select "Just the push event"
6. Click Add webhook

✅ Now, whenever users.txt is modified, Jenkins will trigger the pipeline automatically.


**Jenkinsfile to Process Only New Users**

```
pipeline {
    agent any
    environment {
        VM_LIST = "192.168.1.101 192.168.1.102 192.168.1.103 192.168.1.104 192.168.1.105 192.168.1.106 192.168.1.107 192.168.1.108 192.168.1.109 192.168.1.110"
        USERS_FILE = "users.txt"
    }
    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'git@github.com:your-repo.git'
            }
        }

        stage('Detect New Users') {
            steps {
                script {
                    def lastCommit = sh(script: "git log -1 --pretty=format:'%H'", returnStdout: true).trim()
                    def changedFiles = sh(script: "git diff-tree --no-commit-id --name-only -r ${lastCommit}", returnStdout: true).trim()

                    if (changedFiles.contains(USERS_FILE)) {
                        echo "users.txt updated! Processing new users..."
                        def oldUsers = readFile('.user_cache.txt').trim().split("\n") as Set
                        def currentUsers = readFile(USERS_FILE).trim().split("\n") as Set
                        def newUsers = currentUsers - oldUsers
                        
                        if (newUsers.isEmpty()) {
                            echo "No new users found."
                            currentBuild.result = 'ABORTED'
                            return
                        } else {
                            echo "New users detected: ${newUsers.join(', ')}"
                            writeFile(file: '.user_cache.txt', text: currentUsers.join("\n"))
                            env.NEW_USERS = newUsers.join(" ")
                        }
                    } else {
                        echo "No changes detected in users.txt"
                        currentBuild.result = 'ABORTED'
                        return
                    }
                }
            }
        }

        stage('Grant Access to New Users') {
            steps {
                sshagent(['jenkins-ssh-key']) {
                    script {
                        def userList = env.NEW_USERS.split(" ")
                        def vmList = env.VM_LIST.split(" ")

                        for (vm in vmList) {
                            for (user in userList) {
                                sh """
                                ssh root@${vm} "
                                sudo useradd -m -s /bin/bash ${user} &&
                                echo '${user}:StrongPassword123' | sudo chpasswd &&
                                sudo usermod -aG sudo ${user}"
                                """
                            }
                        }
                    }
                }
            }
        }
    }
}


```






