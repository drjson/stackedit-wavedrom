# Making StackEdit Work With WaveDrom
A user extension hack for StackEdit. [View on StackEdit](https://stackedit.io/viewer#!url=https://raw.github.com/drjson/stackedit-wavedrom/master/README.md).
![Example Screenshot](https://raw.githubusercontent.com/drjson/stackedit-wavedrom/master/screenshot.png "Example Screenshot")

[TOC]

## Overview
I wanted to integrate the excellent [WaveDrom][1] timing diagram tool within the [StackEdit][2] tool for documentation of hardware timing. This documents the required settings and items for getting it to work. Open this document in StackEdit to confirm operation of the plugin by viewing the included examples.

## Implementation
### Theory of Operation
Using the StackEdit user extension, a fenced code block is used which has the type of `wavedrom`. This code block is replaced with a `<script>` block compatible with WaveDrom during the editor output generation. In the case of exporting to HTML template, this is compatible with the WaveDrom `processAll()` function or with the user extension for the preview.

### Current Version and Limitations
Currently the preview and export is working with a custom user extension by utilizing the fencing code blocks with a type of `wavedrom`. Since WaveDrom doesn't have a https:// CDN, you will need to force your browser to allow unsafe scripts for the preview to work. It may require a refresh of the browser window to have the WaveDrom graphs generated. You can also use the templates and view the output in the templated HTML output.

The current WaveDrom codebase does not provide convenient methods for rendering individual graphs and some of the source from the WaveDrom base was required to be copied into the user extension in order to get the live preview to work efficiently.

###Installation
#### User Extension Code
Copy the following code into the user extension area of StackEdit under `Settings->Extensions->UserCustom Extension`.
```javascript
// create <script>
var waveDromScript = document.createElement('script');
waveDromScript.src = 'http://wavedrom.com/WaveDrom.js';

var waveDromTheme = document.createElement('script');
waveDromTheme.src = 'http://wavedrom.com/skins/default.js';

// append <script>
var firstScript = document.getElementsByTagName('script')[0];
firstScript.parentNode.insertBefore(waveDromTheme, firstScript);
firstScript.parentNode.insertBefore(waveDromScript, firstScript);

// Setup parsing
userCustom.onPagedownConfigure = function(editor) {
	var previewContentsElt = document.getElementById('preview-contents');
	editor.hooks.chain("onPreviewRefresh", function() {
	    $(previewContentsElt)
		    .find('.prettyprint > .language-wavedrom')
			.each(function() 
		{
			$(this).replaceWith("<script type=\"WaveDrom\">"
	    	    				  + this.textContent
				    			  + "</script>");
		});
    });
};

// Borrow these snippets from the wavedrom source since they aren't publicly available in the API, but are required to call RenderWaveForm.

var lane = {
    xs     : 10,    // tmpgraphlane0.width
    ys     : 10,    // tmpgraphlane0.height
    xg     : 120,   // tmpgraphlane0.x
    // yg     : 0,     // head gap
    yh0    : 0,     // head gap title
    yh1    : 0,     // head gap
    yf0    : 0,     // foot gap
    yf1    : 0,     // foot gap
    y0     : 5,     // tmpgraphlane0.y
    yo     : 30,    // tmpgraphlane1.y - y0;
    tgo    : -10,   // tmptextlane0.x - xg;
    ym     : 150,    // tmptextlane0.y - y0
    xlabel : 6,     // tmptextlabel.x - xg;
    xmax   : 1,
    scale  : 1,
    head   : {},
    foot   : {}
};

// From eva.js in wavedrom
function eva (text) {
    var TheTextBox, source;

    function erra (e) {
        return { signal: [{ name: ['tspan', ['tspan', {class:'error h5'}, 'Error: '], e.message] }]};
    }

    try { 
	    source = eval('(' + text + ')'); 
	} catch (e) {
		return erra(e); 
	}
    


    if (Object.prototype.toString.call(source) !== '[object Object]') {
        return erra({ message: '[Semantic]: The root has to be an Object: "{signal:[...]}"'});
    }
    if (source.signal) {
        if (Object.prototype.toString.call(source.signal) !== '[object Array]') {
            return erra({ message: '[Semantic]: "signal" object has to be an Array "signal:[]"'});
        }
    } else if (source.assign) {
        if (Object.prototype.toString.call(source.assign) !== '[object Array]') {
            return erra({ message: '[Semantic]: "assign" object hasto be an Array "assign:[]"'});
        }
    } else {
        return erra({ message: '[Semantic]: "signal:[...]" or "assign:[...]" property is missing inside the root Object'});
    }
    return source;
}

// Callback to update the preview pane.
userCustom.onPreviewFinished = function() {
		var index = 0;
		// Need this structure from WaveDrom source

		$('#preview-contents script[type=WaveDrom]').each(function() {
			$(this).parent().children().not('script').remove();
			$(this).parent().append('<div id="WaveDrom_Display_'+index+'"></div>');
			WaveDrom.RenderWaveForm(index, eva(this.innerHTML), 'WaveDrom_Display_', lane);
			index++;
		});
		

};
```

#### Template Output Updates
Add the following to your default template (Settings->Advanced->Default Template) in the `<head>` tag:
```html
<script src="http://wavedrom.com/skins/default.js" type="text/javascript"></script>
<script src="http://wavedrom.com/WaveDrom.js" type="text/javascript"></script>
```

And add the following to the `<body>` tag of the template:
```html
<body onLoad="WaveDrom.ProcessAll()">
```
### Examples
Examples from [WaveDrom tutorials](http://wavedrom.com/tutorial.html).

#### Simple Example
```wavedrom
{ signal : [
  { name: "clk",  wave: "p......" },
  { name: "bus",  wave: "x.34.5x",   data: "head body tail" },
  { name: "wire", wave: "0.1..0." },
  { name: "additional", wave: "0...1.." }
]}
```

#### Spacers and Gaps
```wavedrom
{ signal: [
  { name: "clk",         wave: "p.....|..." },
  { name: "Data",        wave: "x.345x|=.x", data: ["head", "body", "tail", "data"] },
  { name: "Request",     wave: "0.1..0|1.0" },
  {},
  { name: "Acknowledge", wave: "1.....|01." }
]}
```

#### DDR Timing
```wavedrom
{ signal: [
  { name: "CK",   wave: "P.......",                                              period: 2  },
  { name: "CMD",  wave: "x.3x=x4x=x=x=x=x", data: "RAS NOP CAS NOP NOP NOP NOP", phase: 0.5 },
  { name: "ADDR", wave: "x.=x..=x........", data: "ROW COL",                     phase: 0.5 },
  { name: "DQS",  wave: "z.......0.1010z." },
  { name: "DQ",   wave: "z.........5555z.", data: "D0 D1 D2 D3" }
]}
```
#### Headers, Footers, and Indexes
```wavedrom
{signal: [
  {name:'clk', wave: 'p.....PPPPp....' },
  {name:'dat', wave: 'x....2345x.....', data: 'a b c d' },
  {name:'req', wave: '0....1...0.....' }
],
head: {text:
  ['tspan',
    ['tspan', {class:'error h1'}, 'error '],
    ['tspan', {class:'warning h2'}, 'warning '],
    ['tspan', {class:'info h3'}, 'info '],
    ['tspan', {class:'success h4'}, 'success '],
    ['tspan', {class:'muted h5'}, 'muted '],
    ['tspan', {class:'h6'}, 'h6 '],
    'default ',
    ['tspan', {fill:'pink', 'font-weight':'bold', 'font-style':'italic'}, 'pink-bold-italic']
  ]
},
foot: {text:
  ['tspan', 'E=mc',
    ['tspan', {dy:'-5'}, '2'],
    ['tspan', {dy: '5'}, '. '],
    ['tspan', {'font-size':'25'}, 'B '],
    ['tspan', {'text-decoration':'overline'},'over '],
    ['tspan', {'text-decoration':'underline'},'under '],
    ['tspan', {'baseline-shift':'sub'}, 'sub '],
    ['tspan', {'baseline-shift':'super'}, 'super ']
  ],tock:-5
}
}
```
#### Logic Diagrams
```wavedrom
{ assign:[
  ["out",
    ["|",
      ["&", ["~", "a"], "b"],
      ["&", ["~", "b"], "a"]
    ]
  ],
  ["y", ["~", "x"]],
  ["y", ["=", "x"]],
  ["y", ["&", "a" ,"b"]],
  ["y", ["~&", "a", "b"]],
  ["y", ["|", "a", "b"]],
  ["y", ["~|", "a", "b"]],
  ["y", ["^", "a", "b"]],  
  ["y", ["~^", "a", "b"]],
  ["y", ["+", "a", "b"]],
  ["y", ["*", "a", "b"]]
]}
```
[1]: http://www.wavedrom.com "WaveDrom"
[2]: http://www.stackedit.io "StackEdit"