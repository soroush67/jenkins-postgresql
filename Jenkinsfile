// Jenkins Pipeline job: "Pipeline script from SCM" pointing at this
// repo, with Script Path = Jenkinsfile.
//
// "Deploy to any destination" = point INVENTORY_HOST at any host in
// inventory/hosts.ini (or add a new one there first) - no code changes
// needed per target.
//
// One-time Jenkins setup required before this works:
//   1. Set AGENT_NODE_LABEL below to the real label of the agent node
//      that has docker + (docker compose plugin OR the standalone
//      docker-compose binary - roles/preflight detects whichever is
//      present) + ansible-core installed, and has its OS user in the
//      `docker` group.
//   2. Create two "Secret text" credentials - one for the appuser
//      password (ID below: PG_PASSWORD_CREDENTIALS_ID), one for the
//      postgres_exporter password (PG_EXPORTER_PASSWORD_CREDENTIALS_ID).
//      Both are required - roles/preflight refuses to deploy with either
//      one empty.
//   3. Replace inventory/hosts.ini's placeholder localhost entry with
//      your real target host(s) before deploying for real.

def AGENT_NODE_LABEL = 'CHANGE_ME_POSTGRES_AGENT_LABEL'
def PG_PASSWORD_CREDENTIALS_ID = 'pg-stack-appuser-password'
def PG_EXPORTER_PASSWORD_CREDENTIALS_ID = 'pg-stack-exporter-password'

pipeline {
    agent { label AGENT_NODE_LABEL }

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    parameters {
        choice(
            name: 'ACTION',
            choices: ['deploy', 'status'],
            description: 'deploy: idempotent PostgreSQL + postgres_exporter deploy (safe to rerun). status: read-only health/connectivity check.'
        )
        string(
            name: 'INVENTORY_HOST',
            defaultValue: 'localhost',
            description: 'Inventory host (from inventory/hosts.ini) to target - "deploy to any destination" just means changing this, or adding a new host to that file first.'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Deploy') {
            when { expression { params.ACTION == 'deploy' } }
            environment {
                INVENTORY_HOST = "${params.INVENTORY_HOST}"
            }
            steps {
                withCredentials([
                    string(credentialsId: PG_PASSWORD_CREDENTIALS_ID, variable: 'PG_PASSWORD'),
                    string(credentialsId: PG_EXPORTER_PASSWORD_CREDENTIALS_ID, variable: 'PG_EXPORTER_PASSWORD'),
                ]) {
                    sh '''
                        set -e
                        ansible-playbook playbooks/deploy.yml \
                          --limit "$INVENTORY_HOST" \
                          -e pg_password="$PG_PASSWORD" \
                          -e pg_exporter_password="$PG_EXPORTER_PASSWORD"
                    '''
                }
            }
        }

        stage('Status') {
            when { expression { params.ACTION == 'status' } }
            environment {
                INVENTORY_HOST = "${params.INVENTORY_HOST}"
            }
            steps {
                sh 'ansible-playbook playbooks/status.yml --limit "$INVENTORY_HOST"'
            }
        }
    }
}
