pipeline {
    agent any

    parameters {
        string(name: 'REPO_NAME', defaultValue: '', description: 'Enter the name of the new repository')
        string(name: 'USERS', defaultValue: '', description: 'Comma-separated GitHub usernames to grant write access (e.g., user1,user2)')
    }

    environment {
        ORG_NAME = 'nameoftheorg'  // GitHub organization name
        GITHUB_API = 'https://api.github.com'
    }

    stages {
        stage('Create Repository') {
            steps {
                script {
                    env.REPO_NAME = params.REPO_NAME.trim()  // Store in env so it’s accessible across all stages

                    if (!env.REPO_NAME) {
                        error("❌ Repository name cannot be empty!")
                    }

                    withCredentials([string(credentialsId: 'github_pat', variable: 'GITHUB_TOKEN')]) {
                        def createRepoCmd = """
                            curl -s -o response.json -w "%{http_code}" -X POST -H "Authorization: token \${GITHUB_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 ${env.GITHUB_API}/orgs/${env.ORG_NAME}/repos \
                                 -d '{ "name": "${env.REPO_NAME}", "private": true, "description": "Created via Jenkins pipeline", "auto_init": true }'
                        """

                        def httpStatus = sh(script: createRepoCmd, returnStdout: true).trim()
                        def response = readFile('response.json')

                        if (httpStatus == "201") {
                            echo "✅ Repository '${env.REPO_NAME}' created successfully!"
                        } else if (httpStatus == "403") {
                            error("🚫 GitHub API rate limit exceeded! Response: ${response}")
                        } else {
                            error("❌ Failed to create repository. HTTP Status: ${httpStatus}, Response: ${response}")
                        }
                    }
                }
            }
        }

        stage('Add Collaborators') {
            steps {
                script {
                    def users = params.USERS.split(',').collect { it.trim() }.findAll { it }
                    if (users.isEmpty()) {
                        echo "ℹ️ No users provided. Skipping collaborator addition."
                    } else {
                        withCredentials([string(credentialsId: 'github_pat', variable: 'GITHUB_TOKEN')]) {
                            users.each { user ->
                                def addUserCmd = """
                                    curl -s -o response.json -w "%{http_code}" -X PUT -H "Authorization: token \${GITHUB_TOKEN}" \
                                         -H "Accept: application/vnd.github.v3+json" \
                                         ${env.GITHUB_API}/repos/${env.ORG_NAME}/${env.REPO_NAME}/collaborators/${user} \
                                         -d '{ "permission": "push" }'
                                """

                                def httpStatus = sh(script: addUserCmd, returnStdout: true).trim()
                                def response = readFile('response.json')

                                if (httpStatus == "201" || httpStatus == "204") {
                                    echo "✅ Successfully added ${user} as a collaborator!"
                                } else {
                                    error("❌ Error adding ${user}. HTTP Status: ${httpStatus}, Response: ${response}")
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Set Branch Protection Rules') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'github_pat', variable: 'GITHUB_TOKEN')]) {
                        def protectBranchCmd = """
                            curl -s -o response.json -w "%{http_code}" -X PUT -H "Authorization: token \${GITHUB_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 ${env.GITHUB_API}/repos/${env.ORG_NAME}/${env.REPO_NAME}/branches/master/protection \
                                 -d '{
                                    "required_status_checks": null,
                                    "enforce_admins": true,
                                    "required_pull_request_reviews": {
                                        "dismiss_stale_reviews": false,
                                        "require_code_owner_reviews": false,
                                        "required_approving_review_count": 1
                                    },
                                    "restrictions": null
                                 }'
                        """

                        def httpStatus = sh(script: protectBranchCmd, returnStdout: true).trim()
                        def response = readFile('response.json')

                        if (httpStatus == "200") {
                            echo "✅ Branch protection applied to 'master'!"
                        } else if (httpStatus == "403") {
                            error("🚫 GitHub API rate limit exceeded! Response: ${response}")
                        } else {
                            error("❌ Failed to apply branch protection. HTTP Status: ${httpStatus}, Response: ${response}")
                        }
                    }
                }
            }
        }
    }
}
