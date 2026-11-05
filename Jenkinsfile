// Collects commit details for a GitHub event (push / pull_request) and extracts Jira issue keys.
// Triggered by a GitHub webhook via the Generic Webhook Trigger plugin, or run manually
// with a COMMIT_SHA parameter.

pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  parameters {
    string(name: 'COMMIT_SHA', defaultValue: '',
           description: 'Manual runs only: commit to inspect (blank = HEAD of the checked-out branch). Ignored for webhook runs.')
    string(name: 'JIRA_PROJECT_KEY', defaultValue: 'DEMO',
           description: 'Jira project key to look for, e.g. DEMO for DEMO-123')
  }

  triggers {
    GenericTrigger(
      genericVariables: [
        [key: 'gh_ref',       value: '$.ref',                  defaultValue: ''],
        [key: 'gh_before',    value: '$.before',               defaultValue: ''],
        [key: 'gh_after',     value: '$.after',                defaultValue: ''],
        [key: 'gh_repo',      value: '$.repository.full_name', defaultValue: ''],
        [key: 'gh_pusher',    value: '$.pusher.name',          defaultValue: ''],
        [key: 'gh_pr_number', value: '$.pull_request.number',  defaultValue: ''],
        [key: 'gh_pr_action', value: '$.action',               defaultValue: ''],
        [key: 'gh_pr_title',  value: '$.pull_request.title',   defaultValue: ''],
        [key: 'gh_pr_sha',    value: '$.pull_request.head.sha', defaultValue: ''],
        [key: 'gh_pr_branch', value: '$.pull_request.head.ref', defaultValue: ''],
        [key: 'gh_pr_base',   value: '$.pull_request.base.ref', defaultValue: '']
      ],
      causeString: 'GitHub webhook',
      token: 'jira-jenkins-lab',
      printContributedVariables: true,
      printPostContent: false,
      silentResponse: false
    )
  }

  stages {
    stage('Resolve event') {
      steps {
        script {
          if (env.gh_pr_number) {
            env.EV_TYPE       = 'pull_request'
            env.EV_SHA        = env.gh_pr_sha
            env.EV_BRANCH     = env.gh_pr_branch
            env.EV_RANGE_BASE = env.gh_pr_base ? "origin/${env.gh_pr_base}" : ''
            env.EV_LIMIT      = '20'
          } else if (env.gh_after) {
            env.EV_TYPE   = 'push'
            env.EV_SHA    = env.gh_after
            env.EV_BRANCH = (env.gh_ref ?: '').replaceFirst('^refs/(heads|tags)/', '')
            // 'before' is all zeros when a branch is first created
            env.EV_RANGE_BASE = (env.gh_before && !(env.gh_before ==~ /^0+$/)) ? env.gh_before : ''
            env.EV_LIMIT      = '20'
          } else {
            env.EV_TYPE       = 'manual'
            env.EV_SHA        = params.COMMIT_SHA ?: 'HEAD'
            env.EV_BRANCH     = env.BRANCH_NAME ?: ''
            env.EV_RANGE_BASE = ''
            env.EV_LIMIT      = '1'
          }

          // 'after' is all zeros when a branch or tag is deleted: nothing to inspect
          if (env.EV_SHA ==~ /^0+$/) {
            env.EV_SHA = ''
            echo 'Branch/tag deletion event: no commits to inspect.'
            currentBuild.result = 'NOT_BUILT'
          } else {
            currentBuild.description = "${env.EV_TYPE}: ${env.EV_BRANCH ?: '-'} @ ${env.EV_SHA.take(7)}"
          }
        }
      }
    }

    stage('Fetch refs') {
      when { expression { env.EV_SHA } }
      steps {
        sh '''
          set -eu
          git fetch --no-tags origin '+refs/heads/*:refs/remotes/origin/*'
          if [ -n "${gh_pr_number:-}" ]; then
            git fetch --no-tags origin "pull/${gh_pr_number}/head"
          fi
          git cat-file -e "${EV_SHA}^{commit}" 2>/dev/null || git fetch --no-tags origin "${EV_SHA}"
        '''
      }
    }

    stage('Commit details') {
      when { expression { env.EV_SHA } }
      steps {
        sh '''
          set -eu
          OUT=commit-report.txt

          if [ -n "${EV_RANGE_BASE}" ]; then
            RANGE="${EV_RANGE_BASE}..${EV_SHA}"
          else
            RANGE="-n ${EV_LIMIT} ${EV_SHA}"
          fi

          {
            echo "=== GitHub event ==="
            echo "Type:    ${EV_TYPE}"
            echo "Repo:    ${gh_repo:-${GIT_URL:-unknown}}"
            echo "Branch:  ${EV_BRANCH:-n/a}"
            echo "Pusher:  ${gh_pusher:-n/a}"
            if [ -n "${gh_pr_number:-}" ]; then
              echo "PR:      #${gh_pr_number} (${gh_pr_action:-}) ${gh_pr_title:-}"
            fi
            echo "Head:    $(git rev-parse "${EV_SHA}")"
            echo

            echo "=== Commits (${RANGE}) ==="
            git log --date=iso-strict --format='%h  %an <%ae>  %ad%n    %s' ${RANGE}
            echo

            echo "=== Head commit ==="
            git show --stat --format=fuller "${EV_SHA}"
            echo

            echo "=== Files changed ==="
            if [ -n "${EV_RANGE_BASE}" ]; then
              git diff --name-status "${EV_RANGE_BASE}" "${EV_SHA}"
            else
              git show --name-status --format= "${EV_SHA}"
            fi
            echo

            echo "=== Jira keys (${JIRA_PROJECT_KEY}-*) ==="
            KEYS=$( { git log --format=%B ${RANGE}; echo "${EV_BRANCH}"; echo "${gh_pr_title:-}"; } \
                    | grep -oE "${JIRA_PROJECT_KEY}-[0-9]+" | sort -u || true )
            if [ -n "${KEYS}" ]; then echo "${KEYS}"; else echo "(none found)"; fi
          } > "${OUT}"

          cat "${OUT}"
        '''
        archiveArtifacts artifacts: 'commit-report.txt', allowEmptyArchive: true
      }
    }
  }
}
