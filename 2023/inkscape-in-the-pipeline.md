# The pipeline
## Main Concept
We covered the rabbit hole (inx) extensively but see how things are now piping together. The basic structure should be quite clear now

![In the pipeline](inkscape-in-the-pipeline/pipeline.svg)

The concept is quite clear now. Before we focus on inkex, let's wrap are head around what this arrow actually means. First of all, the script is ran with python, but which python is that. Well turns out that inkscape in windows ships with python build-in, located at

```cmd
"%programfiles%\Inkscape\bin\python"
```

## Manually running the plugin
Let try to run our script with node.js as a middle man. If we run our script with this interpreter in the extensions map, we can actually take a peak under the hood

```js
const path = require('path');
const { execSync } = require("child_process");

let inkPath = path.join(process.env.ProgramFiles,"Inkscape");
let pythonPath = path.join(inkPath,"bin","python")

execSync(`"${pythonPath}"`, {input:`print('in/out')`,encoding:"utf8"})
```

Ok let's run our script with the interpreter
```js
let script = "kitimo_golden_ratio.py"
let scriptPath = path.join(inkPath,"share","inkscape","extensions",script)

let output = execSync(`"${pythonPath}" "${scriptPath}" -h`, {input:`print('in/out')`,encoding:"utf8"})
console.log(output)
```

This is the result
```
usage: kitimo_golden_ratio.py [-h] [--output OUTPUT] [--tab TAB] [--max MAX]
                              [--r R] [--l L] [--x X] [--y Y] [--c C]
                              [--id IDS] [--selected-nodes SELECTED_NODES]
                              [INPUT_FILE]

positional arguments:
  INPUT_FILE            Filename of the input file (default is stdin)

options:
  -h, --help            show this help message and exit
  --output OUTPUT       Optional output filename for saving the result
                        (default is stdout).
  --tab TAB
  --max MAX
  --r R
  --l L
  --x X
  --y Y
  --c C
  --id IDS              id attribute of object to manipulate
  --selected-nodes SELECTED_NODES
                        id:subpath:position of selected nodes, if any
```
Interesting enough you can see now how the pipeline works, a file is passed (or via stdin or via a real file, default stdout), manipulation is done and then outputted to a file (stdout or a file, default stdout).

```js
let fakeStdIn = `<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!-- Created with Inkscape (http://www.inkscape.org/) -->
<svg
   width="210mm"
   height="297mm"
   viewBox="0 0 210 297"
   version="1.1"
   id="svg5"
   inkscape:version="1.2.1 (9c6d41e410, 2022-07-14)"
   sodipodi:docname="plain.svg"
   xmlns:inkscape="http://www.inkscape.org/namespaces/inkscape"
   xmlns:sodipodi="http://sodipodi.sourceforge.net/DTD/sodipodi-0.dtd"
   xmlns="http://www.w3.org/2000/svg"
   xmlns:svg="http://www.w3.org/2000/svg">
  <sodipodi:namedview
     id="namedview7"
     pagecolor="#ffffff"
     bordercolor="#666666"
     borderopacity="1.0"
     inkscape:showpageshadow="2"
     inkscape:pageopacity="0.0"
     inkscape:pagecheckerboard="0"
     inkscape:deskcolor="#d1d1d1"
     inkscape:document-units="mm"
     showgrid="false"
     inkscape:zoom="0.73851712"
     inkscape:cx="207.17191"
     inkscape:cy="128.63615"
     inkscape:window-width="1920"
     inkscape:window-height="1009"
     inkscape:window-x="-8"
     inkscape:window-y="-8"
     inkscape:window-maximized="1"
     inkscape:current-layer="layer1" />
  <defs
     id="defs2" />
  <g
     inkscape:label="Layer 1"
     inkscape:groupmode="layer"
     id="layer1" />
</svg>
`
let output = ""
output = execSync(`"${pythonPath}" "${scriptPath}" --r 150 --c rgba(32,32,32,255)`, {input:fakeStdIn,encoding:"utf8"})
console.log(output)
```

Yields
```xml
...
<g inkscape:label="golden-circles"><circle inkscape:label="c1" r="150.0" cx="0.0" cy="0.0" style="fill:none;stroke:rgb(32, 32, 32);stroke-width:1;stroke-linejoin:round;paint-order:markers fill stroke;stop-color:#000000"/>
...
```

## Dummy driver
To analyze further, let's make a dummy driver called Kitimo Dummy. You can find it on [Kitimo Dummy](https://github.com/tdewin/kitimo)

It doesn't do anything except create a group and some logging code to "c:\\d\\kitimodummydriver.log" (make sure c:\\d exists before trying or modify the code)

```python
if __name__ == "__main__":
    f = open("c:\\d\\kitimodummydriver.log", "a")
    f.write("{0} - Dummy Test\n".format(current_milli_time()))
    argcmd = ' '.join(sys.argv[0:])
    f.write("{0}\n".format(argcmd))
    
    potentialsvg = sys.argv[len(sys.argv)-1]
    if re.search("[.]svg.*",potentialsvg):
        f.write("Got input svg {0}, dumping..".format(potentialsvg))
        svgfile = open(potentialsvg, "r")
        data = svgfile.read()
        svgfile.close()
        f.write(data)
    else:
        f.write("No sigar {0} ...".format(potentialsvg))
        
    f.close()
    KitimoDummy().run()
```

When we run it, you can see the following 
```
1678825109581 - Dummy Test
kitimo_dummy.py --tab=Parameters --l=1 --r=1 C:\Users\Timothy\AppData\Local\Temp\ink_ext_XXXXXX.svg9PYW11
Got input svg C:\Users\Timothy\AppData\Local\Temp\ink_ext_XXXXXX.svg9PYW11, dumping..<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!-- Created with Inkscape (http://www.inkscape.org/) -->

<svg
   width="210mm"
   height="297mm"
   viewBox="0 0 210 297"
   version="1.1"
   id="svg5"
...
```
What is a bit bizarre imho is that it seems that the input is done via tmp file but the output is going to stdout since there is no --output parameter. 

# Conclusion
The extensions/effects just gets the whole svg files via a tmp file, edits it and output on stdout. Manipulation is done via the parameters, eg --r --c --l.