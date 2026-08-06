# Allure report host

The published report is on the [`gh-pages`](../../tree/gh-pages) branch and is
served at <https://amielnoy.github.io/allure-report-learn-practice-deploy/>.

`main` holds only the deployment workflow. It is deliberately kept clear of the
report: the publishing job in `LearnPracticeWork` force-pushes `gh-pages` with
`keep_files: false`, which would delete a workflow stored alongside the report.
