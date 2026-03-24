# Code review

Hey Ram, thanks for the PR! I've got a few specific suggestions for ways I think this code could be improved which I'll write below, but feel free to ping me if you want to chat it through it any more detail, or if anything doesn't makes sense. 🙂

### General feedback

Firstly, here's some general feedback:

- **Comments**: You've included a lot of comments in your code. For me, this suggests that the code itself is not clear enough on its own if it requires lots of explanation alongside it. Think about if any of the comments are redundant (basically just a translation of the code below it - one example of this is your `Init arrays` comment), or how you could improve the code to make it more obvious what it is doing. The code should speak for itself.

- **Validation ordering**: You've included a good few edgecases for error handling, which is great to see. However, I think the way these are structured in the endpoint could be better. For example, I'd expect to see some more request shape validation at the top - at the moment, it is assumed that the request body will be an array. The `.filter` and `.entries` methods will fail if this isn't the case, so it would be prudent to include some checks on this, and return a `400` if the body is not an array (or handle it differently). Similarly, you don't throw your error for the duplicated username until after you've run `Student.create` - meaning that here you are accessing the database even after you know it will fail. I'd recommend restructuring this to avoid unnecessary database writes, i.e. get the validation out of the way before you touch the database. Disentangling the validation logic from the database logic should also help make it a bit clearer to read, as it's currently all a little bit jumbled up. 

- **Testing:** I'd ideally like to see some automated tests covering this new endpoint, including capturing the edgecases that result in the various error states. You could have also considered including some testing notes in the PR description with instructions for how reviewers can test this locally to check it works.

- **Scaling**: With a max student number of 50 per request, this code will cope OK. However, I wonder if we should be thinking about the scalability of this code. The fact that it has database writes for each student in the array (one `Student.create` per student) is something that may not scale very well  - for example, if we get to 1000s of students per request. There are some things we could consider implementing in order to improve this - such as chunking (bulk inserts), or implementing a more asyncronous approach (using queues/background jobs) rather than waiting for one upload after the other. This would be fine for a proof of concept, however!


### Feedback on specific lines of code:

1. You're using CommonJS `require` syntax in the imports here, which isn't the most modern way to write Javascript anymore.

```javascript
const Student = require('../models/student')
```
 Most modern codebases use the ES Modules syntax which would be:

```javascript
import Student from '../models/student'
```
Feel free to ignore this comment if the rest of the codebase is using `CommonJS` 😅 The main thing I'd be worried about here is consistency with everything else (CommonJS is still fully supported after all).

---

2. You've set `preflight` to default to `true`. Given the function is called `createStudents` I would have expected this to mean that the default mode is creation, rather than validation. Could you consider changing the default, or perhaps renaming the function?

```javascript
const createStudents = async (req, res, preflight = true) => {
```

---

3. The ``${index}.`` part of this line doesn't seem right to me, since all you're doing is concatenating a `.` onto `index` - perhaps this is a typo?  I'd expect just `index` to be passed directly instead:

```diff
- formatValidationErrors(error, `${index}.`, { username: student.username })
+ formatValidationErrors(error, index, { username: student.username })
```

---

4. I think `maxStudents` here will end up being a string instead of a number if there is a value in the `MAX_STUDENTS_PER_REQUEST` env variable.

```javascript
const maxStudents = process.env.MAX_STUDENTS_PER_REQUEST || 50
```

This means the `count > maxStudents` check might behave oddly. It would be better to explictly validate that `maxStudents` is a number, perhaps by using `parseInt`. 

---

5. There are a few little syntax tidy-ups that I spotted thanks to a VSCode extension I use called [P42](https://marketplace.visualstudio.com/items?itemName=p42ai.refactor). I'd really recommend installing it - I've learned a lot through using it regularly. Some of the things it picked up on:
- `{ transaction: transaction, discardPassword: preflight }` - `transaction` can be collapsed into just the one instance of the word.
- This code block doesn't need to be nested inside an else block - it can just sit after the `if` block without needing curly braces. 

```diff
-       else {
        await transaction.commit()
        res.status(201).json({ created: students })
-       }
```
- `return studentErrors['errors']` can be rewritten as `return studentErrors.errors`

---

6. Did you mean to include the actual `schoolId` in the 404 response? I think this would be a useful thing to include instead of the string `"schoolId"`, to help us debug when a school is unexpectedly not found:
```diff
 res.status(404).json([
        {
-          path: 'schoolId',
+          path: req.params.schoolId,
```

---

7. For the "too many students" error, I'd probably expect the error location to be 'body' instead of 'path', since the issue is with the body payload.

```diff
 res.status(400).json([
        {
          path: '.',
          errorCode: 'ERR_TOO_MANY_STUDENTS',
          message: `Can not create more than ${maxStudents} students in one request.`,
-          location: 'path',
+          location: 'body',
        },
      ])
```

---

8. I would probably rename the `students` array to `studentIds` since this more accurately describes what it is. Perhaps you could even distinguish between `verifiedStudentIds` and `createdStudentIds` - this would make it really easy to see what is returned in the response bodies at a glance. 

---

9. Your duplicate name check currently runs through the entire array of students, for each student, which is quite inefficient. It would be quite an easy-win for scalability if we moved that username check outside the loop. Instead, we could count how many of each username there is upfront (just one calculation across the list of students), and then refer to this in the loop without having to do the calculation again. It could look something like:

```javascript
const usernameCounts = {}

for (const student of req.body) {
    usernameCounts[student.username] = (usernameCounts[student.username] || 0) + 1
}

// then check if there is a duplicate inside the loop:
const usernameIsDuplicated = usernameCounts[student.username] > 1

if (usernameIsDuplicated) {
    // error logic
}
```

---

10. As the code stands, you'll also need to include `await transaction.rollback()` in the outermost `catch` block (line 124). Currently if you end up in there you risk leaving a transaction open. 
