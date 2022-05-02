# Introduction

**To any rogue Github explorer**, this file was created for a singular person whom I promised to help with a project. If you are not that person, nothing interesting to see here. I hope you enjoy your Github discovery session :)

And for that person, here we go:

## Some words

Hello there! I am sorry it could not be a productive session because I got stuck on a very simple thing. I have decided to create this file for you so that I can further explain what the further steps are. **Everything I put here is in chronological order**, so that you can commit them whenever you wish. I will try to add as much explanation as I can, but feel free to ask if you need help!

Of course, change whatever you want. I don't want to do your assignment. But use as much ideas from this as you want!

Note: If I put ellipsis (...) somewhere in the code, there is probably a large chunk of code I did not change. I recommend double-checking everything and run the code after every 1-2 code blocks. That way you can identify errors earlier. I made sure that everything works at the end of every section so you can go in bulks as well.

# The code

## 1. The And/Or Table Structure

I have thought a lot about what the best way to approach this would be, and I could finally come up with an idea. The issue we were having was with prerequisites such as "(A or B) and (C or D)". I figured adding an extra "group id" could fix the issue. Here is what Or Table would look like:

| Primary Key | Course  | Prerequisite  | Group ID  |
|---|---|---|---|
| 1  | 1  | 2  | 1  |
| 2  | 1  | 3  | 1  |
| 3  | 1  | 4  | 2  |
| 4  | 1  | 5  | 2  |

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

It was becoming really difficult to keep track of anything in the admin page, so I added \_\_str\_\_ functions following function inside OrTable and AndTable definitions. This way you can clearly see the group, prerequisite and course of both databases' objects Such that:

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

**DO NOT FORGET TO MIGRATE THE DATABASE!!**

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

This was genuinely painful. I left comments to explain how the code works but do not hesitate to ask me how it works!

### homepage/templates/homepage/available.html
```html
<h3>Available courses</h3>
<ul>
{% for course in available_courses %}
  <li>{{ course }}</li>
{% endfor %}
</ul>
```

Notice that there was no change for imports or home() in the following code:

### homepage/views.py
```python
...
def can_take_course(course_id, selected_ids):
    """
    Take a course ID, and IDs of selected courses, return whether the selected
    courses satisfy as prerequisites for the course ID
    """
    # Get the "and" prerequisites
    and_list = set(
        AndTable.objects.filter(course=course_id).values_list(
            "prereq", flat=True
        )
    )
    if and_list:
        # Return false if one of these prerequisites are selected
        for course in and_list:
            if course not in selected_ids:
                return False
    # Get the "or" prerequisites
    or_list = OrTable.objects.filter(course=course_id)
    if or_list:
        # Get the groups (1, 2, 3, ...) for this or selection
        groups = set(or_list.values_list("group_id", flat=True))
        for group in groups:
            # For every group, get a set of courses
            or_set = set(
                OrTable.objects.filter(
                    course=course_id, group_id=group
                ).values_list("prereq", flat=True)
            )
            # If no courses from this list is selected,, return falase
            if not or_set.intersection(selected_ids):
                return False
    # If all "false" conditions fail, return True
    return True


def available(request):
    available_courses = []
    if request.method == "POST":
        # Get IDs of all courses
        all_course_ids = CourseDataBase.objects.all().values_list(
            "id", flat=True
        )
        # Get IDs of the courses selected in the previous page
        selected_ids = [
            get_object_or_404(CourseDataBase, pk=course_id).id
            for course_id in request.POST.getlist("courses")
        ]
        # Decide if the student can take them
        can_take = [
            can_take_course(int(course), set(selected_ids))
            for course in all_course_ids
        ]
        # Get the names of available courses
        available_courses = [
            CourseDataBase.objects.get(id=course_id).name
            for i, course_id in enumerate(all_course_ids)
            if can_take[i]
        ]
    context = {"available_courses": available_courses}
    return render(request, "homepage/available.html", context)

```

You can later alter the last function so that you pass more information about the courses!

## 4. Making things pretty

I will switch to some CSS work because the last function has burnt all gears in my brain :/ Before that though, let's make a header for the webpage. Create the file "homepage/templates/homepage/header.html". I am keeping it separate because we do not want to copy paste the same thing to both of our views.

### homepage/templates/homepage/header.html

```html
<header>
  <h1 id="header-text" class="centered">AUC Course Thingy</h1>
</header>
```

Now, add the following code in the beginning of other two HTML files. That will get the HTML contents from header.html and put it there.

### homepage/templates/homepage/available.html AND homepage/templates/homepage/index.html

```html
{% block header %}
   {% include "./header.html" %}
{% endblock %}

```

I have also added some class and IDs to Index HTML to prepare for styling:

### homepage/templates/homepage/index.html

```html
{% block header %}
   {% include "./header.html" %}
{% endblock %}

<div id="content" class="centered">
  <p>Select the courses you have already taken</p>
  <form method='post' action="available/">
    {% csrf_token %}
    {% if courses %}
      {% for course in courses %}
        <input class="checkbox" type="checkbox" name="courses" value="{{course.id}}" >
        <label class="checkbox-label" for="courses"> {{course.name}} </label><br>
      {% endfor %}
    {% else %}
      <p> No courses available </p>
    {% endif %}
  <input class="button" type="submit" value="Send">
  </form>
</div>
```

Now it's time to finally work on CSS! There are multiple things to do. First, of course, you create a static file. (These are fines that will never be affected by Django. CSS files, images and audio files are examples for that.) The file should be at "homepage/static/homepage/style.css"

### homepage/static/homepage/style.css

```css
* {
    border: 1px solid red;
}
```

For every page you want to "include" these stylings, you need to have a <link> tag somewhere, plus the "load static" command for Django. Since you have a header in every page already, I recommend simply placing this at the top of your header.html file:

### homepage/templates/homepage/header.html

```html
{% load static %}
<link rel="stylesheet" type="text/css" href="{% static 'homepage/style.css' %}">
...
```

You should not see any changes yet. **Every time you add or remove a static file, you need to stop and restart the server.** Once you restart once, it should pick up changes automatically. If it does not, force-refresh the webpage with Ctrl+F5 or Ctrl+R or whatever Mac alternatice there is for that. If you do that now, you should see that everything has red border around them!

Okay, now is the time to make stuff actually pretty. Go back to "homepage/static/homepage/style.css" and **delete everything inside it.**

I will start with changing the font because I really dislike the default fonts for the browser. Let's go for something more modern. I often get my fonts from [Google Fonts](https://fonts.google.com/).

### homepage/static/homepage/style.css

```css
@import url('https://fonts.googleapis.com/css2?family=Ubuntu&display=swap');

html {
    font-family: "Ubuntu";
}
```

Then, I would like to center the elements. The dot "." implies that we are working with a class. The hash "#" implies that it's an id.

```css
...
.centered {
    margin-left: 15%;
}
```

I will also change the header to be the red color of AUC, adding margin and paddings, while changing the font color. Lastly, I will work on the content itself. The final CSS file should look something like this:

```css
@import url('https://fonts.googleapis.com/css2?family=Ubuntu&display=swap');

* {
    margin: 0;
}

:root {
    --dark: #8b0000;
    --light: #fee;
}

html {
    font-family: "Ubuntu";
}

.centered {
    margin-left: 15%;
}

header {
    background-color: var(--dark);
    color: var(--light);
    padding: 20px 0;
}

#content {
    margin-top: 30px;
}

#content .checkbox {
    width: 50px;
    accent-color: var(--dark);
}

#content .checkbox, li, .warning {
    margin-top: 10px
}

.button {
    margin-top: 40px;
    border: none;
    padding: 10px;
    cursor: pointer;
    border-radius: 10px;
}
.button:hover {
    background-color: var(--dark);
    color: var(--light);
}
```

If you have any questions about the CSS, let me know!

Oh, I realized I forgot about available.html, so here it is:

### homepage/templates/homepage/available.html

```html
{% block header %}
   {% include "./header.html" %}
{% endblock %}

<div id="content" class="centered">
  <p>Available courses</p>
  <ul>
  {% for course in available_courses %}
    <li>{{ course }}</li>
  {% endfor %}
  </ul>
</div>

```

I will go back to styling but it looks fine for now. Let's get more functionality done:

## 5. Making available.html better

It's long overdue, so let's add a go back button:

### homepage/templates/homepage/available.html

```html
{% block header %}
   {% include "./header.html" %}
{% endblock %}

<div id="content" class="centered">
  <p>Available courses</p>
  <ul>
  {% for course in available_courses %}
    <li>{{ course }}</li>
  {% endfor %}
  </ul>

  <form action="../">
    <input type="submit" value="Back" class="button"/>
  </form>
</div>
```

The first idea on my mind is to separate the courses you have already selected from the list of courses you can take. This will require a lot of work again, in multiple files. I will start with views.py:

### homepage/views.py
```python
...
def available(request):
    available_courses = []
    if request.method == "POST":
        # Get IDs of all courses
        all_course_ids = CourseDataBase.objects.all().values_list(
            "id", flat=True
        )
        # Get IDs of the courses selected in the previous page
        selected_ids = [
            get_object_or_404(CourseDataBase, pk=course_id).id
            for course_id in request.POST.getlist("courses")
        ]
        # Decide if the student can take them
        can_take = [
            can_take_course(int(course), set(selected_ids))
            for course in all_course_ids
        ]
        # Get the names of available courses
        available_courses = [
            CourseDataBase.objects.get(id=course_id).name
            for i, course_id in enumerate(all_course_ids)
            if can_take[i]
            if CourseDataBase.objects.get(id=course_id).id not in selected_ids
        ]
        # Get the names of already taken courses
        selected_courses = [
            CourseDataBase.objects.get(id=course_id).name
            for course_id in selected_ids
        ]
    context = {
        "available_courses": available_courses,
        "selected_courses": selected_courses,
    }
    return render(request, "homepage/available.html", context)
```

Visually nothing has changed. Let's work on it now:

### homepage/templates/homepage/available.html
```html
{% block header %}
   {% include "./header.html" %}
{% endblock %}

<div id="content" class="centered">
  <p>Available courses</p>
  <ul>
  {% for course in available_courses %}
    <li>{{ course }}</li>
  {% endfor %}
  </ul>
  <br>
  <p>Courses already taken</p>
  <ul>
  {% for course in selected_courses %}
    <li>{{ course }}</li>
  {% endfor %}
  </ul>
  <form action="../">
    <input type="submit" value="Back" class="button"/>
  </form>
</div>
```

The very last thing I want to do here is this: Imagine someone clicking on Submit on homepage with no selected values. Then the "selected values" part will be empty. Similarly, if you select everything, then the available courses section will be empty. I want it to display some text instead:

### homepage/templates/homepage/available.html
```html
{% block header %}
   {% include "./header.html" %}
{% endblock %}

<div id="content" class="centered">
  <p>Available courses</p>
  {% if available_courses %}
    <ul>
    {% for course in available_courses %}
      <li>{{ course }}</li>
    {% endfor %}
    </ul>
  {% else %}
    <p class="warning"><i>No more courses available!</i></p>
  {% endif  %}
  <br>
  <p>Courses already taken</p>
  {% if selected_courses %}
    <ul>
    {% for course in selected_courses %}
      <li>{{ course }}</li>
    {% endfor %}
    </ul>
  {% else %}
    <p class="warning"><i>No courses were selected!</i></p>
  {% endif  %}
  <form action="../">
    <input type="submit" value="Back" class="button"/>
  </form>
</div>
```

# What now?

## Notes

That is all I could do before I got tired of it and I hope it is helpful for you too! Don't hesitate to ask any questions. If you want to meet again to go over this and potentially add more stuff, let me know!

## Ideas to go forward

Here is a list of ideas if you want to work further on this:

- Show more information such as majors, level and credit on the list. You may need to use table tags (and therefore td, tr and th) instead of the unordered list we use. You will also need to do some editing on views.py to pass relevant data.
- Handle long data. I tested this with 10 courses but I know you will have a lot more. If you do not want people to scroll down to reach the back button, you can use the CSS property called "overflow". If you give a div some height and say "overflow-y: auto", then it will become scrollable when its content is overflowing. Something to look into :/
- Still it's overwhelming to see all options together, so some way to filter the selection could be helpful. For example, a dropdown with 3 majors to display only those majors is more than enough!
