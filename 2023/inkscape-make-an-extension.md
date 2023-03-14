# What are Inkscape extensions
They are basically plugins for Inkscape that can be made under the Extension menu in python. In essence, they are scripts that are passed input and generate SVG.

The Base directory for extensions in windows is 
```
%ProgramFiles%\Inkscape\share\inkscape\extensions
```

On *nix I would expect something like 
```
/usr/share/inkscape/extensions
```

For generating a template we will be using node.js, but you can get a generated Golden Circle from [tdewin/kitimo](https://github.com/tdewin/kitimo). In that case, you might have to manually rename some items. The Golden circle extension just create circle following the fibonacci row. You can the use this excellent tutorial from [Logos By Nick](https://www.youtube.com/watch?v=1hhAXrxVMeU). Honestly, this guy is the best Inkscape resource out there.

# Getting started with node
The extension folder itself is in program files and thus requires you to start node.js as admin in order to add the script files (Right click node and select Run as administrator, even from the start menu).

Now, to get started with node, it is good to load some generic libs to write and create paths. Using path library should mean that once you have the extPath properly setup, this could also work in *nix.

```node
const path = require('path');
const fs = require('fs');
```

Let's first check if path exists (you might need to update dependent on your env). This should say "true". If it doesn't, then it is probably not going to work.
```node
let extPath = path.join(process.env.ProgramFiles,"Inkscape","share","inkscape","extensions");
fs.existsSync(extPath);
```

# Making base files to manipulate
Let's make a plugin framework, let's first make some default variables. These are not really Inkscape variables, just some naming I came up to make sure the template is consistent.

* effectMenu is the grouping menu under extensions
* plugin menu is basically the function
* uid is something you might adapt to your own domain/github to make it unique

```node
let effectMenu = "Kitimo"
let pluginName = "Golden Ratio"
let fullName = `${effectMenu} ${pluginName}`
let fName = fullName.replaceAll(" ","_").toLowerCase()
let className = fullName.replaceAll(" ","")
let uid = `com.github.tdewin.inkscape.${className}`
```

I personally like this javascript multiline backticks format because you can then "execute code within your template with ${}". 

# Making the menu
Now let's make a menu. Inkscape has it own markup language to define the UI. This is stored in a separate inx file. It is almost like an HTML file in that it purely defines a GUI in XML. It does not have any real logic and is purely there to define the input.

In this tutorial we will only pass int (integer, non decimal numbers), but actually inx allows you to define more complex types like color. This is passed the scope here. Here we will just have 2 tabs, one that defines the input parameters, and one that defines the help.

Interesting enough you can see at the bottom that it refers to a python file. Theoretically, inkscape can work with other languages but I've only seen samples with python.

See [inx-overview](https://inkscape.gitlab.io/extensions/documentation/authors/inx-overview.html) and [inx-parameters](https://inkscape.gitlab.io/extensions/documentation/authors/inx-widgets.html#parameters)
```node
let menuFile = path.join(extPath,`${fName}.inx`)
let menuInx = `<?xml version="1.0" encoding="UTF-8"?>
<inkscape-extension xmlns="http://www.inkscape.org/namespace/inkscape/extension">
    <name>${pluginName}</name>
    <id>${uid}</id>
    <param name="tab" type="notebook">
        <page name="Parameters" gui-text="Main Parameters">
            <param name="max" type="int" min="2" max="50"   gui-text="Number of circles">8</param>
            <param name="r"   type="int" min="1" max="100" gui-text="R of circle 1">1</param>
            <param name="l"   type="int" min="1" max="100" gui-text="Line of circle">1</param>
            <param name="x"   type="int" min="-10000" max="10000" gui-text="Offset X">0</param>
            <param name="y"   type="int" min="-10000" max="10000" gui-text="Offset Y">0</param>
        </page>
        <page name="Help" gui-text="Help">
            <label xml:space="preserve">This extension is just a sample to get started. Creates golden ratio's circle according to Fibonacci numbers
            </label>
        </page>
    </param>
    <effect>
        <object-type>all</object-type>
        <effects-menu>
            <submenu name="${effectMenu}"/>
        </effects-menu>
    </effect>
    <script>
        <command location="inx" interpreter="python">${fName}.py</command>
    </script>
</inkscape-extension>`

fs.writeFileSync(menuFile,menuInx,"utf8")
console.log("Wrote to ",menuFile)
```

Now let's make some small code sample. Seems that there are different kind of base extensions. Theoretically we will be generating circles (not modifying an existing object) so initially I thought to extend GenerateExtension. But this is basically some template over EffectExtension that did a bunch of stuff I didn't like. For example, I wanted to be in full control of the transform functionality. It did help me to understand how to make groups, attach to the layer and use the transform functionality so not completely useless.

The way it works is that you inherit the EffectExtension and then override 2 methods:
* add_arguments : basically tells python how to parse the input. This should match in inx parameters. Your parameters will be stored in self.options. Eg --r will be parsed to self.options.r
* effect : do something. I basically had to go in the code to figure out that there is a directory with elements. With those base elements you can create elements. I've also added transformations and styling. This is again, not documented but tried to reverse engineer it from the Inkscape source code linked below

Finally, inkscape use "inkscape:label" as a semi replacement for id's in that sense, "inkscape:label" do not have to be unique in your svg, id do have to be. Outside of inkscape, id's might make more sense but again they have to be unique. Using the inkex API seems to autogenerate those IDs.

```node
let pyFile = path.join(extPath,`${fName}.py`)
let code = `#!/usr/bin/env python
# coding=utf-8
# MIT 2023 github.com/tdewin

import inkex

from inkex.elements import TextElement,Group,Circle
from inkex.transforms import Transform
from inkex.styles import Style

class ${className}(inkex.EffectExtension):
    def add_arguments(self, pars):
        pars.add_argument("--tab")
        pars.add_argument("--max", type=int, default=8)
        pars.add_argument("--r", type=int, default=1)
        pars.add_argument("--l", type=int, default=1)
        pars.add_argument("--x", type=int, default=0)
        pars.add_argument("--y", type=int, default=0)


    def effect(self):
        layer = self.svg.get_current_layer()
        
        g = Group.new("golden-circles")

        fib = [1,2]
        max = self.options.max-len(fib)
        for x in range(max):
            l = len(fib)
            fib.append(fib[l-1]+fib[l-2])

        ### If you wanted to print text with the circles
        # textElement = TextElement()
        # textElement.set("inkscape:label","textGolden")
        # textElement.set("id","textGolden")
        # textElement.text = ', '.join(str(x) for x in fib)
        # g.append(textElement)

        r = self.options.r
        s = Style.parse_str("fill:none;stroke:#333333;stroke-width:{0};stroke-linejoin:round;paint-order:markers fill stroke;stop-color:#000000".format(self.options.l))

        
        for f in fib:
            c = Circle()
            c.set("inkscape:label","c{0}".format(f))
            c.radius = r*f
            # no offset, we global offset with transform on the group
            # but just for documentation purposes
            c.center = [0,0]
            c.style = s
            g.append(c)

        
        
        g.transform = Transform(translate=(self.options.x, self.options.y))

        layer.append(g)

      

if __name__ == "__main__":
    ${className}().run()
`

fs.writeFileSync(pyFile,code,"utf8")
console.log("Wrote to ",pyFile)
```


# External links
* https://inkscape.org/develop/extensions/
* [inx-overview](https://inkscape.gitlab.io/extensions/documentation/authors/inx-overview.html)
* [inx-parameters](https://inkscape.gitlab.io/extensions/documentation/authors/inx-widgets.html#parameters)
* https://inkscape-extensions-guide.readthedocs.io/en/latest/02_hello-world.html
* [initpy](https://gitlab.com/inkscape/extensions/-/blob/master/inkex/elements/__init__.py)
* https://gitlab.com/inkscape/extensions/-/blob/master/inkex/extensions.py
* [polygons; circles, rectangles](https://gitlab.com/inkscape/extensions/-/blob/master/inkex/elements/_polygons.py)
* [style](https://gitlab.com/inkscape/extensions/-/blob/master/inkex/styles.py)