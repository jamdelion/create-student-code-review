# Disclosure

This is how I worked on this task, and outlines the extent to which I used AI (ChatGPT enterprise).

### Method:
- Go through the code manually at first, writing notes on issues noticed
- Copied the code into my code editor locally - I rely on VSCode plugins to spot some syntax issues.
- Passed the code through an AI agent (ChatGPT) for additional thoughts
- Googled some things I wasn't sure about - e.g. preflight, database savepoints, `findByPk`
- Wrote up my code review in markdown without AI help.

---

Here's a breakdown of what thoughts I had myself, and what came from AI.

### Initial thoughts:

Firstly, what is the context of this piece of code? I.e. what is the experience level of the person who wrote the code? Is this a proof of concept or production ready code? Is it a codebase with a lot of legacy/old syntax that we are fitting into, or very new/cutting edge?

- Old 'require' syntax (context dependent, but modern usage is import?)
- Are big comments necessary? Could this be formatted as a docstring instead? I think code should be able to speak for itself without needing lots of commenting
- what is `findByPk`? Could this be more readable/better named? (find by id perhaps)
- Res.status errorCode --> should this be an enum pointing to a number (404)?
- Could split out some of the logic into functions, e.g. the max_students section
- Lots of unneeded comments, e.g. 'Init arrays', 'do this via a Sequelize ValidationError`. The code should speak for itself.
- Generally quite long and hard to read, could be organised better, e.g. validation first.
- `${index}.` seems like a typo, could just pass index (string interpolates to just the index with a dot)
- `maxStudents` might be a string if reading from process.env var? So then the count > maxStudents will fail. (parseInt needed)
- path: 'schoolId', presumably you want the real schoolId here so you know which id isn't working.
- collapse transaction: transaction, remove unnecessary `else`. (mention p42 vscode extension)
- tests? Or way to test that the code is working etc
- typescript could help

### Comments from ChatGPT

- Validation mode is default, which feels wrong for a 'createStudents' function. 
- filter and entries methods assumes the request body is an array. Expect request shape validation before this point. Reject non-array bodies with a 400.
- duplicate username detection runs for every student in the array (won't scale well). Better to precompute username count map before the loop? E.g use a set
- transaction might be left open - rollback in the outercatch if transaction exists.
- static savepoint name - maybe a bit fragile
- location: 'body' might be better for too many students error (issue is in the body payload, not the URL path)
- we create the student even if duplicatedUsername, i.e we're creating model instances even when we know it will fail
- rename students to studentIds/verifiedStudentIds
