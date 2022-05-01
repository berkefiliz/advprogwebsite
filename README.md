## Introduction

Hello there! I am sorry it could not be a productive session because I got stuck on a very simple thing. I have decided to create this file for you so that I can further explain what the further steps are. **Everything I put here is in chronological order**, so that you can commit them whenever you wish. I will try to add as much explanation as I can, but feel free to ask if you need help!

Of course, change whatever you want. I don't want to do your assignment. But use as much ideas from this as you want!

## 1. The And/Or Table Structure

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
- **If you already have a lot of OrTable rows, you will need to do a migration.** If there are 2-3 entries, the effort is not worth it. Just delete the entries, copy the code, then run it. You can find migration stuff on [Django Tutorials](https://docs.djangoproject.com/en/4.0/intro/tutorial02/). **ALSO NOTICE THE CHANGE FROM PROTECT TO CASCADE!**

### homepage/models.py
```python
...
class OrTable(models.Model):
    prereq = models.ForeignKey(
        CourseDataBase, on_delete=models.CASCADE, related_name="prereq_or"
    )
    course = models.ForeignKey(
        CourseDataBase, on_delete=models.CASCADE, related_name="course_or"
    )
    group_id = models.IntegerField(default=1)

class AndTable(models.Model):
    prereq = models.ForeignKey(
        CourseDataBase, on_delete=models.CASCADE, related_name="prereq_and"
    )
    course = models.ForeignKey(
        CourseDataBase, on_delete=models.CASCADE, related_name="course_and"
    )
```

### vterm (bash)
```bash
python manage.py makemigrations homepage
python manage.py migrate
```

## 2. Displaying actual information on url/homepage/available

The next steps will require a lot of testing, so I wanted the page to be visually more informative. First of all, I changed the HTML to list every available course. Here, the *ul* tag refers to an *unordered list* and *li* tag refers to a *list item*.

The code below should display all selected courses into a list. Of course, **they are only numbers so far!**

### homepage/templates/homepage/available.html
```html
<h3>Available courses</h3>
<ul>
{% for course in selected %}
  <li>{{ course }}</li>
{% endfor %}
</ul>
```

In order to turn "course" into text, we should pass a list of strings to the view. We will do it by sending queries to Course Database. Before we do that, we would like to ensure the selected object exists no matter what. To do so, we just need to define it before reading the POST data:

### homepage/views.py
```python
...
def available(request):
    selected_courses = []
    if request.method == "POST":
        selected_courses = request.POST.getlist("courses")
    context = {"selected": selected_courses}
    return render(request, "homepage/available.html", context)
```

Now we can do the requests for the database. (Be aware of the new import!) (More explanation below)

### homepage/views.py
```python
from django.shortcuts import get_object_or_404, render
...
def available(request):
    selected_courses = []
    if request.method == "POST":
        selected_courses = [
            get_object_or_404(CourseDataBase, pk=course_id).name
            for course_id in request.POST.getlist("courses")
        ]
    context = {"selected": selected_courses}
    return render(request, "homepage/available.html", context)
```

Explanation of above function:
1. I initialize selected_courses so that I am guaranteed to have a list no matter.
2. I then check if the method is POST. This is a way to chech whether the reason we are on this page is through clicking a form with "post" method.
3. For all of the courses we get from that post action (request.POST.getlist("courses")), I get the name of the CourseDataBase object with the appropriate course ID. The "courses" comes from "homepage/templates/homepage/index.html" where all checkboxes have the name "courses".


## 3. Calculating the courses that you can take, given the list of courses you have taken

Before everything, you will realize the admin page is a bit difficult to read. For human readibility, I added the following function inside OrTable and AndTable definitions. Such that:

### homepage/models.py
```python
class OrTable(models.Model):
    prereq = models.ForeignKey(
        CourseDataBase, on_delete=models.PROTECT, related_name="prereq_or"
    )
    course = models.ForeignKey(
        CourseDataBase, on_delete=models.PROTECT, related_name="course_or"
    )
    group_id = models.IntegerField(default=1)

    def __str__(self):
        return f"[{self.group_id}] {self.course.name} <-- {self.prereq.name}"
        
class AndTable(models.Model):
    prereq = models.ForeignKey(
        CourseDataBase, on_delete=models.CASCADE, related_name="prereq_and"
    )
    course = models.ForeignKey(
        CourseDataBase, on_delete=models.CASCADE, related_name="course_and"
    )

    def __str__(self):
        return f"{self.course.name} <-- {self.prereq.name}"
```

Now it will be easier to track what line belongs to what prerequisite and what its group id is.
