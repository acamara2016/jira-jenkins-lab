/*
 * Release-notes pipeline
 *
 * On every run it:
 *   1. checks out the code and shows the commit details,
 *   2. builds a version  <BASE_VERSION>-<UTC timestamp>   e.g. 1.0.0-20261104233015,
 *   3. collects every commit since the last successful build and the Jira tickets they mention,
 *   4. writes release-notes-<version>.md (archived on the build),
 *   5. publishes to Jira: creates the version, sets "Fix versions" on every ticket and comments on it.
 *
 * Jira connection (see jenkins.yaml / .env.example):
 *   env  JIRA_BASE_URL     https://<your-site>.atlassian.net
 *   env  JIRA_PROJECT_KEY  default project key (can be overridden with the job parameter)
 *   cred jira-api          "Username with password" = Jira account email + API token
 */

def release = [:]   // state shared between stages
def scmVars = [:]

pipeline {
    agent any

    options {
        skipDefaultCheckout(true)          // we run `checkout scm` ourselves to get the git variables
        disableConcurrentBuilds()          // two runs at once would race on "since last build"
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timeout(time: 15, unit: 'MINUTES')
    }

    parameters {
        string(name: 'BASE_VERSION', defaultValue: '1.0.0',
               description: 'Version prefix. Release version = <BASE_VERSION>-<yyyyMMddHHmmss in UTC>')
        string(name: 'JIRA_PROJECT_KEY', defaultValue: '',
               description: 'Jira project key. Leave empty to use the JIRA_PROJECT_KEY environment variable (DEMO if unset).')
        booleanParam(name: 'PUBLISH_TO_JIRA', defaultValue: true,
                     description: 'Untick for a dry run: build the release notes but do not create anything in Jira.')
        booleanParam(name: 'SINCE_LAST_BUILD', defaultValue: true,
                     description: 'Only include commits since the last successful build. Untick to include the last MAX_COMMITS commits.')
        string(name: 'MAX_COMMITS', defaultValue: '50',
               description: 'How many commits to include on the first build, or when SINCE_LAST_BUILD is unticked.')
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    scmVars = checkout(scm)
                }
            }
        }

        stage('Version') {
            steps {
                script {
                    release.projectKey = (params.JIRA_PROJECT_KEY ?: env.JIRA_PROJECT_KEY ?: 'DEMO').trim().toUpperCase()
                    if (!(release.projectKey ==~ /[A-Z][A-Z0-9_]*/)) {
                        error("Invalid Jira project key '${release.projectKey}'")
                    }

                    def baseVersion = (params.BASE_VERSION ?: '1.0.0').trim()
                    def ts = sh(returnStdout: true, script: 'date -u +%Y%m%d%H%M%S').trim()
                    release.timestamp   = ts
                    release.releaseDate = "${ts.substring(0, 4)}-${ts.substring(4, 6)}-${ts.substring(6, 8)}"
                    release.releasedAt  = "${release.releaseDate} ${ts.substring(8, 10)}:${ts.substring(10, 12)}:${ts.substring(12, 14)} UTC"
                    release.version     = "${baseVersion}-${ts}"

                    currentBuild.displayName = "#${env.BUILD_NUMBER} ${release.version}"
                    echo "Release version: ${release.version}"
                }
            }
        }

        stage('Commit Details') {
            steps {
                script {
                    release.branch   = (env.BRANCH_NAME ?: scmVars.GIT_BRANCH ?: 'HEAD').replaceFirst('^origin/', '')
                    release.sha      = gitOut('rev-parse HEAD')
                    release.shortSha = gitOut('rev-parse --short HEAD')

                    // Which commits go into this release?
                    def maxCommits = toInt(params.MAX_COMMITS, 50)
                    def prev = scmVars.GIT_PREVIOUS_SUCCESSFUL_COMMIT
                    def usePrev = flag(params.SINCE_LAST_BUILD, true) && prev &&
                            sh(returnStatus: true, script: "git cat-file -e '${prev}^{commit}'") == 0
                    def logRange
                    if (usePrev) {
                        logRange = "${prev}..HEAD"
                        release.rangeLabel = "since last successful build (${prev.substring(0, 7)}..${release.shortSha})"
                    } else {
                        logRange = "--max-count=${maxCommits} HEAD"
                        release.rangeLabel = flag(params.SINCE_LAST_BUILD, true) ?
                                "up to the last ${maxCommits} commits (no previous successful build)" :
                                "up to the last ${maxCommits} commits (SINCE_LAST_BUILD is off)"
                    }

                    def raw = sh(returnStdout: true, script:
                            "#!/bin/sh -e\ngit log ${logRange} --date=short " +
                            "--format='%H%x1f%h%x1f%an%x1f%ae%x1f%ad%x1f%s%x1f%b%x1e'")
                    release.commits = parseCommits(raw, release.projectKey)

                    // Everything below is the "show commit details" part
                    echo '================ HEAD commit ================'
                    sh 'git show --stat --format=fuller --no-color HEAD'
                    echo "================ Commits in this release: ${release.commits.size()} (${release.rangeLabel}) ================"
                    echo release.commits ? commitTable(release.commits) : 'No new commits since the last successful build.'

                    // Diff summary for the same range
                    release.diffStat = ''
                    if (release.commits) {
                        // parent of the oldest commit in the release; the empty tree if that commit is the root
                        def base = gitOut("rev-parse --verify --quiet ${release.commits[-1].sha}^")
                        if (!base) { base = gitOut('hash-object -t tree /dev/null') }
                        release.diffStat = gitOut("diff --shortstat ${base} HEAD")
                        if (release.diffStat) { echo "Diff: ${release.diffStat}" }
                    }

                    // Jira keys: from commit messages + the branch name (feature/DEMO-1-hello)
                    def keys = []
                    for (c in release.commits) {
                        for (k in c.tickets) { if (!keys.contains(k)) { keys.add(k) } }
                    }
                    for (k in findIssueKeys(release.branch, release.projectKey)) {
                        if (!keys.contains(k)) { keys.add(k) }
                    }
                    release.ticketKeys = sortKeys(keys)
                    echo "Jira tickets referenced: ${release.ticketKeys ? release.ticketKeys.join(', ') : '(none)'}"
                }
            }
        }

        stage('Jira Lookup') {
            steps {
                script {
                    release.jiraBase = (env.JIRA_BASE_URL ?: '').trim().replaceAll('/+$', '')
                    release.tickets = []

                    if (!release.jiraBase) {
                        if (flag(params.PUBLISH_TO_JIRA, true)) {
                            error('JIRA_BASE_URL is not set. Set it in .env (see .env.example) or untick PUBLISH_TO_JIRA.')
                        }
                        echo 'JIRA_BASE_URL not set - release notes will list ticket keys without Jira details.'
                        for (k in release.ticketKeys) {
                            release.tickets.add([key: k, found: null, url: '', type: '', status: '', summary: ''])
                        }
                        return
                    }

                    def project = jiraRequest(release.jiraBase, 'GET', "/rest/api/3/project/${release.projectKey}")
                    if (project.status == 401 || project.status == 403) {
                        error("Jira rejected the credentials (HTTP ${project.status}). Check JIRA_USER_EMAIL / JIRA_API_TOKEN behind the 'jira-api' credential.")
                    }
                    if (project.status == 404) {
                        error("Jira project '${release.projectKey}' not found at ${release.jiraBase}.")
                    }
                    if (project.status != 200) {
                        error("Unexpected Jira response (HTTP ${project.status}) while reading project ${release.projectKey}: ${project.content}")
                    }
                    def projectJson = readJSON(text: project.content)
                    release.projectId   = projectJson.id.toString()
                    release.projectName = projectJson.name.toString()
                    echo "Jira project: ${release.projectName} (${release.projectKey}, id ${release.projectId})"

                    for (k in release.ticketKeys) {
                        def resp = jiraRequest(release.jiraBase, 'GET', "/rest/api/3/issue/${k}?fields=summary,issuetype,status")
                        if (resp.status == 200) {
                            def issue = readJSON(text: resp.content)
                            release.tickets.add([
                                key    : k,
                                found  : true,
                                url    : "${release.jiraBase}/browse/${k}",
                                type   : issue.fields.issuetype.name.toString(),
                                status : issue.fields.status.name.toString(),
                                summary: issue.fields.summary.toString()
                            ])
                        } else if (resp.status == 404) {
                            echo "WARNING: ${k} is mentioned in a commit but does not exist in Jira - it will not be linked."
                            release.tickets.add([key: k, found: false, url: '', type: '', status: '', summary: ''])
                        } else {
                            unstable("Could not read ${k} from Jira (HTTP ${resp.status}).")
                            release.tickets.add([key: k, found: null, url: "${release.jiraBase}/browse/${k}", type: '', status: '', summary: ''])
                        }
                    }
                }
            }
        }

        stage('Release Notes') {
            steps {
                script {
                    release.notesFile = "release-notes-${release.version}.md"
                    release.notes = buildReleaseNotes(release, env.BUILD_NUMBER, env.BUILD_URL)
                    sh 'rm -f release-notes-*.md'   // the workspace is reused: archive only this build's notes
                    writeFile(file: release.notesFile, text: release.notes)

                    echo '================ RELEASE NOTES ================'
                    echo release.notes

                    def ticketList = release.tickets.collect { it.key }.join(', ')
                    currentBuild.description = "${release.version}: ${release.commits.size()} commits" +
                            (ticketList ? " | ${ticketList}" : '')
                }
            }
        }

        stage('Publish to Jira') {
            when { expression { flag(params.PUBLISH_TO_JIRA, true) } }
            steps {
                script {
                    def base = release.jiraBase
                    def linkedTickets = release.tickets.findAll { it.found == true }

                    // 1) the Jira version (this is what Jira's "Releases" page / release notes are built from)
                    def description = "Jenkins build #${env.BUILD_NUMBER} (${release.branch} @ ${release.shortSha}): " +
                            "${release.commits.size()} commits, ${release.tickets.size()} tickets" +
                            (release.tickets ? " (${release.tickets.collect { it.key }.join(', ')})" : '')
                    if (description.length() > 250) { description = description.substring(0, 247) + '...' }

                    def created = jiraRequest(base, 'POST', '/rest/api/3/version', [
                        name       : release.version,
                        description: description,
                        projectId  : Integer.parseInt(release.projectId),
                        released   : true,
                        releaseDate: release.releaseDate
                    ])
                    if (created.status != 201) {
                        error("Could not create Jira version ${release.version} (HTTP ${created.status}): ${created.content}")
                    }
                    release.versionId  = readJSON(text: created.content).id.toString()
                    release.versionUrl = "${base}/projects/${release.projectKey}/versions/${release.versionId}"
                    echo "Created Jira version ${release.version} -> ${release.versionUrl}"

                    // 2) tag every ticket with the version and leave a short note on it
                    for (t in linkedTickets) {
                        def fix = jiraRequest(base, 'PUT', "/rest/api/3/issue/${t.key}", [
                            update: [fixVersions: [[add: [name: release.version]]]]
                        ])
                        if (fix.status == 204 || fix.status == 200) {
                            echo "${t.key}: Fix version set to ${release.version}"
                        } else {
                            unstable("${t.key}: could not set Fix version (HTTP ${fix.status}): ${fix.content}")
                        }

                        def commitsForTicket = release.commits.findAll { it.tickets.contains(t.key) }
                        def note = jiraRequest(base, 'POST', "/rest/api/3/issue/${t.key}/comment",
                                adfComment(release.version, release.versionUrl, "Jenkins build #${env.BUILD_NUMBER}",
                                           env.BUILD_URL, commitsForTicket))
                        if (note.status == 201) {
                            echo "${t.key}: release comment added"
                        } else {
                            unstable("${t.key}: could not add comment (HTTP ${note.status}): ${note.content}")
                        }
                    }

                    currentBuild.description = (currentBuild.description ?: release.version) + " | ${release.versionUrl}"
                    echo "Jira release: ${release.versionUrl}"
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'release-notes-*.md', allowEmptyArchive: true
        }
    }
}

// ---------------------------------------------------------------------------------------------
// helpers
// ---------------------------------------------------------------------------------------------

// Run a git command, return trimmed stdout, never fail the build.
String gitOut(String args) {
    return sh(returnStdout: true, script: "#!/bin/sh -e\ngit ${args} 2>/dev/null || true").trim()
}

boolean flag(def value, boolean dflt) {
    return value == null ? dflt : value.toString() == 'true'
}

int toInt(def value, int dflt) {
    try {
        return Integer.parseInt(value.toString().trim())
    } catch (NumberFormatException e) {
        return dflt
    }
}

// Call the Jira Cloud REST API. Returns the httpRequest response; the caller checks .status.
def jiraRequest(String base, String method, String path, Map body = null) {
    def args = [
        url               : base + path,
        httpMode          : method,
        authentication    : 'jira-api',
        acceptType        : 'APPLICATION_JSON',
        validResponseCodes: '100:599',
        timeout           : 30,
        quiet             : true
    ]
    if (body != null) {
        args.contentType = 'APPLICATION_JSON'
        args.requestBody = writeJSON(returnText: true, json: body)
    }
    return httpRequest(args)
}

// Jira keys (e.g. DEMO-12) for one project found in free text. Case-insensitive, returned upper-case.
@NonCPS
List<String> findIssueKeys(String text, String projectKey) {
    def found = []
    def m = java.util.regex.Pattern.compile('(?i)(?<![A-Za-z0-9])(' + projectKey + '-\\d+)(?!\\d)').matcher(text ?: '')
    while (m.find()) {
        def key = m.group(1).toUpperCase()
        if (!found.contains(key)) { found.add(key) }
    }
    return found
}

// Sort keys by issue number (DEMO-2 before DEMO-10). Hand-rolled because the sandbox rejects List.sort().
@NonCPS
List<String> sortKeys(List<String> keys) {
    def sorted = []
    for (k in keys) {
        def n = Integer.parseInt(k.substring(k.lastIndexOf('-') + 1))
        def i = 0
        while (i < sorted.size() && Integer.parseInt(sorted[i].substring(sorted[i].lastIndexOf('-') + 1)) <= n) { i++ }
        sorted.add(i, k)
    }
    return sorted
}

// Turn the raw `git log` output (fields separated by \x1f, records by \x1e) into a list of maps.
@NonCPS
List parseCommits(String raw, String projectKey) {
    def commits = []
    for (rec in raw.split('\u001e')) {
        def f = rec.split('\u001f', -1)
        if (f.length < 6 || f[0].trim() == '') { continue }
        def body = f.length > 6 ? f[6].trim() : ''
        commits.add([
            sha    : f[0].trim(),
            short  : f[1],
            author : f[2],
            email  : f[3],
            date   : f[4],
            subject: f[5],
            body   : body,
            tickets: findIssueKeys(f[5] + '\n' + body, projectKey)
        ])
    }
    return commits
}

@NonCPS
String commitTable(List commits) {
    return commits.collect { c ->
        "${c.short}  ${c.date}  ${c.author.padRight(20).substring(0, 20)}  ${c.subject}" +
                (c.tickets ? "  [${c.tickets.join(', ')}]" : '')
    }.join('\n')
}

@NonCPS
String mdEscape(String s) {
    return (s ?: '').replace('|', '\\|').replace('\n', ' ')
}

@NonCPS
String commitLine(Map c) {
    return "- `${c.short}` ${c.subject} (${c.author}, ${c.date})"
}

@NonCPS
String buildReleaseNotes(Map release, String buildNumber, String buildUrl) {
    def commits = release.commits
    def tickets = release.tickets
    def out = []

    out.add("# Release ${release.version}")
    out.add('')
    out.add('| | |')
    out.add('|---|---|')
    out.add("| **Project** | ${release.projectKey}${release.projectName ? ' - ' + release.projectName : ''} |")
    out.add("| **Build** | [#${buildNumber}](${buildUrl}) |")
    out.add("| **Branch** | ${release.branch} |")
    out.add("| **Commit** | `${release.shortSha}` |")
    out.add("| **Released** | ${release.releasedAt} |")
    out.add("| **Changes** | ${commits.size()} commits, ${release.rangeLabel}${release.diffStat ? ' - ' + release.diffStat.trim() : ''} |")
    out.add('')

    out.add("## Jira tickets (${tickets.size()})")
    out.add('')
    if (tickets) {
        out.add('| Ticket | Type | Status | Summary |')
        out.add('|---|---|---|---|')
        for (t in tickets) {
            def link = t.url ? "[${t.key}](${t.url})" : t.key
            def summary = t.found == false ? '_not found in Jira_' : mdEscape(t.summary)
            out.add("| ${link} | ${mdEscape(t.type)} | ${mdEscape(t.status)} | ${summary} |")
        }
    } else {
        out.add('_No Jira tickets referenced._')
    }
    out.add('')

    out.add('## Changes by ticket')
    out.add('')
    def untracked = commits.findAll { it.tickets.isEmpty() }
    def any = false
    for (t in tickets) {
        def cs = commits.findAll { it.tickets.contains(t.key) }
        if (cs.isEmpty()) { continue }
        any = true
        out.add("### ${t.key}${t.summary ? ' - ' + mdEscape(t.summary) : ''}")
        for (c in cs) { out.add(commitLine(c)) }
        out.add('')
    }
    if (!any) {
        out.add('_No commits reference a Jira ticket._')
        out.add('')
    }

    out.add('## Other changes (no Jira ticket)')
    out.add('')
    if (untracked) {
        for (c in untracked) { out.add(commitLine(c)) }
    } else {
        out.add('_None._')
    }
    out.add('')

    if (commits.isEmpty()) {
        out.add('> No new commits since the last successful build.')
        out.add('')
    }
    return out.join('\n')
}

// Jira Cloud comments use Atlassian Document Format (ADF).
@NonCPS
Map adfComment(String version, String versionUrl, String buildLabel, String buildUrl, List commits) {
    def content = []
    content.add([type: 'paragraph', content: [
        [type: 'text', text: 'Included in release '],
        [type: 'text', text: version, marks: [[type: 'link', attrs: [href: versionUrl]]]],
        [type: 'text', text: ' - '],
        [type: 'text', text: buildLabel, marks: [[type: 'link', attrs: [href: buildUrl]]]]
    ]])
    if (commits) {
        content.add([type: 'bulletList', content: commits.collect { c ->
            [type: 'listItem', content: [[type: 'paragraph', content: [
                [type: 'text', text: c.short, marks: [[type: 'code']]],
                [type: 'text', text: " ${c.subject} - ${c.author}"]
            ]]]]
        }])
    }
    return [body: [type: 'doc', version: 1, content: content]]
}
