You will review some existing code bases before starting work.

- ../handbook : for the "prreview" tool
- ../gh-pulls-summary : overall

The tool I want to write will do a summary of all pull requests in a given repo on github.com or github.

Command line arguments:
- repo URL (default: github.com)
- repo owner (default: current repo's owner)
- repo name (default: current repo's name)
- start timestamp (default: 1 year ago)
- end timestamp (default: now)

The output will be a CSV containing the information per PR.

- when was the PR created
- when was the PR last marked ready for review
- how many comments on the PR (including in-line comments)
- how many requests for change
- how many approvals
- current PR status (draft, closed, ready for review, merged, etc)

You will write some documents before starting on this tool.

- docs/requirements.md : all the functional requirements and technical requirements
- docs/unit-test-plan.md : high level test plan, see existing docs in gh-pulls-summary
- CONTEXT.md : generated LLM context to help you and future LLMs do work

You can include suggested requirements and unit tests.  In implementation you will strictly stay to the requirements documented.  You will split requirements by "functional" and "non-functional" requirements.  You will not make up requirements, such as having some target response time for the git APIs which we have no control over.

If you have questions for me write them in QUESTIONS.md and make sure I know it exists.

Do not write any code if there are unanswered questions!
