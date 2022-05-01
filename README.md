## Introduction

Hello there! I am sorry it could not be a productive session because I got stuck on a very simple thing. I have decided to create this file for you so that I can further explain what the further steps are. **Everything I put here is in chronological order**, so that you can commit them whenever you wish. I will try to add as much explanation as I can, but feel free to ask if you need help!

Of course, change whatever you want. I don't want to do your assignment. But use as much ideas from this as you want!

## The And/Or Table Structure

I have thought a lot about what the best way to approach this would be, and I could finally come up with an idea. The issue we were having was with prerequisites such as "(A or B) and (C or D)". I figured adding an extra "group id" could fix the issue. Here is what Or Table would look like:

| Primary Key | Course  | Prerequisite  | Group ID  |
|---|---|---|---|
| 1  | 1  | 2  | 1  |
| 2  | 1  | 3  | 1  |
| 3  | 1  | 4  | 2  |
| 3  | 1  | 5  | 2  |

Notice how A and B have the Group ID 1 while C and D have the Group ID 2. At the same time, these are prerequisites for the same exact course.

The implementation should be as folowing:
- I created a new column named group_id
- I set the default value to 1, so that you do not have to fill it in every time.
- **If you already have a lot of OrTable rows, you will need to do a migration.** If there are 2-3 entries, the effort is not worth it. Just delete the entries, copy the code, then run it. You can find migration stuff on [Django Tutorials](https://docs.djangoproject.com/en/4.0/intro/tutorial02/).

```python
# homepage/models.py

...
class OrTable(models.Model):
    prereq = models.ForeignKey(
        CourseDataBase, on_delete=models.PROTECT, related_name="prereq_or"
    )
    course = models.ForeignKey(
        CourseDataBase, on_delete=models.PROTECT, related_name="course_or"
    )
    group_id = models.IntegerField(default=1)
...
```
