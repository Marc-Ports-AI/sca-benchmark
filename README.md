# SCA benchmark corpus

Three repositories with pinned dependency sets, a register of every planted
component, and the advisories expected from a scan of each baseline. All
advisory data is attributed against a pinned snapshot of the reviewed GitHub
Advisory Database (commit 524f896ffaae2a2b42c61096d6c94d8e3984edcd,
2026-07-06T22:55:38+00:00).

## Extraction

Windows (PowerShell):

    Expand-Archive -Path sca-benchmark.zip -DestinationPath .

Windows (Explorer): right click the archive and choose Extract All.

macOS or Linux:

    unzip sca-benchmark.zip

Extraction produces a single sca-benchmark directory. Keep its internal
layout intact; push.sh and the tools resolve paths relative to their own
location. On Windows, run the shell scripts from Git Bash or WSL. If a
script is not marked executable after extraction, invoke it through bash:

    bash push.sh <node-remote-url> <python-remote-url> <java-remote-url>

## Layout

    poc-node-sca/       npm corpus: package.json plus committed package-lock.json
    poc-python-sca/     PyPI corpus: pyproject.toml plus pinned requirements.txt
    poc-java-sca/       Maven corpus: pom.xml plus dependency submission workflow
    ground-truth/       register, expected findings, inventories, snapshot pin
    patches/            scenario branch content, one patch per repository
    tools/              attribution and verification scripts
    push.sh             repository initialization and push

Only the three repository directories are pushed to the evaluation org.
Keep ground-truth, patches, and tools in a private location; they are the
adjudication material, not scan input.

## Prerequisites

Git with user.name and user.email configured, and push access to the
evaluation org. Nothing else is needed to publish: the npm lockfile is
committed, the Python pins are static, and the scenario branch content is
pre-generated in patches/, so npm, pip, and Maven are not required on the
publishing machine. Maven 3.x and JDK 11 or later are needed only on
whichever machine runs tools/verify_maven_tree.sh.

## Publishing the repositories

1. Create three empty private repositories in the evaluation org named
   poc-node-sca, poc-python-sca, and poc-java-sca. Do not let the platform
   initialize them with a README or license; the first push must be a
   fast forward.
2. From the sca-benchmark directory, run:

       ./push.sh git@github.com:ORG/poc-node-sca.git \
                 git@github.com:ORG/poc-python-sca.git \
                 git@github.com:ORG/poc-java-sca.git

   HTTPS remote URLs work equally well.

For each repository the script creates one baseline commit on main, tags it
sca-baseline-v1, creates the branch scenario/sca-02-critical-introduction
by applying the matching patch, and pushes the branch, the tag, and main.
Commits carry the identity of whoever runs the script. The script refuses
to run against a directory that already contains .git; remove that
directory to rebuild from a clean state.

## Repository configuration

Enable dependency graph, Dependabot alerts, and Dependabot security updates
on each repository. On poc-java-sca, confirm that the workflow at
.github/workflows/dependency-submission.yml completed at least once on
main; GitHub resolves Maven transitives only through the dependency
submission API, and Maven results are meaningless before that run
finishes. For tools other than GitHub, onboard the three repositories
through the vendor's standard integration and scan the sca-baseline-v1 tag.

## Maven resolution check

Six Maven rows in the register are tagged pending-resolve because their
versions come from the Spring Boot 2.6.6 BOM rather than a committed
lockfile. After the first clone, run from inside poc-java-sca:

    ../tools/verify_maven_tree.sh

Each row prints PASS or FAIL against the expected version. Update any row
that differs in ground-truth/component_register.csv and
ground-truth/expected_findings.csv before treating the Maven counts as
final.

## Baseline scan and adjudication

Scan the sca-baseline-v1 tag with the tool under test and export the
findings. Adjudicate against ground-truth/expected_findings.csv: 146 live
expected findings across the three repositories (35 npm, 40 PyPI, 71
Maven), plus 3 withdrawn exclusions and 17 components verified clean at the
snapshot.

Counting rules. A finding is the tuple (repository, package, version,
advisory). Duplicate install paths of the same tuple count once; the npm
tree contains qs 6.7.0 at two paths for this reason. Advisories withdrawn
at the snapshot are excluded in both directions: reporting one is not an
error, missing one is not a miss. All dependencies are runtime scope. Each
repository is scored on its own; results roll up only at the category
level.

Recall is detected live findings over the per-ecosystem totals, reported
separately for direct and transitive relations. Any finding a tool raises
on an expected-clean row, or on the manifest-floor row (node-fetch), is a
false positive candidate for review. Remediation guidance is scored against
the fix_clears_all column: log4j-core advice must reach 2.17.1 or later,
not stop at 2.15.0. Column definitions and the full tag vocabulary are in
ground-truth/README.md.

## Pull request gate

Open a pull request from scenario/sca-02-critical-introduction into main on
each repository and record the gate behavior. The branch introduces vm2
3.9.19 (npm), pillow 8.1.0 (PyPI), and commons-collections 3.2.1 (Maven).
Expected results are in ground-truth/scenario_sca02_expected.csv: 62 live
advisories in total (31 npm, 29 PyPI, 2 Maven). Score whether the gate
fires, what it blocks or warns on, and whether the suggested remediation
reaches a version that clears the introduced advisories. Close the pull
request without merging when the observations are recorded.

## Reuse across tool windows

The same tag, register, and scenario branch carry over unchanged to each
tool window; onboard the next tool against the identical repositories and
repeat the two procedures above. During the consolidation week, rerun every
tool against the sca-baseline-v1 tag on the same day, and check for
advisory database drift by re-running tools/osv_match.py against a fresh
clone of github/advisory-database. Differences between the pinned snapshot
and the fresh clone explain finding deltas that are database growth rather
than tool behavior.
