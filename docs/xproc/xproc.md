#XProc in eXist
_written by Erik Siegel (erik at xatapult.nl) in March 2014_

## Introduction
 
XProc is a programming language for defining XML document processing pipelines. Many document processing scenarios involve some combination of XML technologies; canonical examples include XInclude, schema validation, and transformations. XProc has been designed specifically to allow authors to compose XML processes and share these compositions in a standard way.

For instance, an example of a simple XProc processing pipeline is processing an XML document by first resolving the XInclude calls, transform it with several consecutive XSL stylesheets, and then validate it against an XML Schema. When it is valid, it is stored to disk. More complex XProc pipelines could involve splitting and merging documents, working with zip files (through extensions), making processing decisions based on XPath (2.0) expressions, etc.

There are several implementations of XProc, but the most widely used is probably Calabash. eXist currently supports XProc using Calabash through its xmlcalabash extension module. This is an experimental implementation and does not by far unleash all XProc’s protential. The same is true for a module called xproc that tries to implement XProc using XQuery.

To amend this and add XProc to the suite of X technologies supported by eXist, a new XProc module was developed in 2013/2014.

## Installation and removal

The XProc module requires eXist 3.0.

### Installation

Install it via eXist's Dashboard > Package Manager. 

### Removal

Uninstalling the package is not the easy normal package manager’s routine for this. Because the package includes a jar file, it is (partly) locked by eXist and some extra work is required:

1. Uninstall the package first through the Package Manager. This seems to work, but when you refresh the list the package still seems to be there.
1. Stop eXist 
1. Navigate to your data directory (by default: *$EXIST_HOME/webapp/WEB-INF/data*) and from there to the package repository directory *expathrepo*.
1. Remove the *xproc-x.y.z* directory
1. Restart eXist
1. When you re-open the package manager, the XProc module should be gone.

You’ll probably need to use this explicit uninstall procedure before you install a new version. Because of the jar file locking, the normal package manager’s effortless update procedure will not work properly.

## Basic usage

Here is a very simple “Hello World” XProc in eXist: 

```xquery
import module namespace xproc = "http://exist-db.org/xproc";

let $simple-xproc as document-node() := document {
    <p:declare-step xmlns:p="http://www.w3.org/ns/xproc" version="1.0">
        <p:input port="source">
            <p:inline>
                <doc>Hello world!\</doc>
            </p:inline>
        </p:input>
        <p:output port="result"/>
        <p:identity/>
    </p:declare-step>
}

return
    xproc:process($simple-xproc)
```

In the above example the actual XProc script is embedded in the XQuery code as a document node and passed as such to the `xproc:process` function. There are however several other possibilities ways to execute an XProc script:

1. You could pass it as an *element()* (pass the root *p:declare-step* element)
1. You could store it in a document in the database and pass it by absolute URI, e.g. *xmldb:exist:///db/path/to/my/xproc.xpl*
1. You could pass the document by relative URI. This will be resolved against the location of the XQuery script.

Simply calling *xproc:process* will start Calabash and do the trick. The function’s result will be the result passed on the pipeline’s primary output port.

## xproc:process

The module exposes three `xproc:process` variants:

1. `xproc:process( $xproc-document )`
  * See the “basic usage” section above.
1. `xproc:process( $xproc-document, $options )`
  * This allows you to pass data for input ports, catch output port results and pass processing op tions to the XProc processor.
1. `xproc:process( $xproc-document, $primary-input, $options )`
  * This allows you to pass the primary input document in an easy way and processing options.
 
The processing options argument `$options` is a sequence of elements, so you can pass multiple processing options. For instance:

```xquery
xproc:process(
    $xproc, (
        <option name="someoption" value="somevalue"/>,
        <input type="xml" port="extrainputport" url="input.xml"/>
    )
)
```
  
### Options

You can pass values for XProc options to the pipeline using:

```xml
<option name="option-name" value="option-value"/>
```

You are limited to string values. Options in a namespace are not supported.

### Connecting input ports

You can connect the input ports to documents (only to full documents, not to XML fragments):

```xml
<input type="xml | data"
    port="portname"
    url="url"/>
```

* When `type="xml"` the document pointed to by the `url` attribute must be a well-formed XML document. It can be absolute or relative. Relative URLs are resolved against the location of the XQuery script.
* When `type="data"` the document pointed to by the `url` attribute will be available to the XProc processor base64-encoded, wrapped in a `<c:data>` element. For unknown reasons, `type="data"` does not accept relative URLs.

Example:
```xml
<input type="xml"
    port="source"
    url="xmldb:exist:///db/path/to/my/xmldocument.xml"/>
```

### Connecting output ports

Besides the results of the primary output port (which is returned as the function’s result), you can also catch the results of other output ports:

```xml
<output port="portname" url="url"/>
```

Note:

* The URL must be an absolute URL pointing to a location in the database
* **Watch out:** The URL must begin with the prefix *xmldb://* (instead of the usual *xmldb:exist://*)
* When the collection the URL points to does not exist or cannot be written to, an exception will be raised. 

Example:

```xml
<output port="extraresult" url="xmldb:///db/path/to/my/data.xml"/>
```

## Additional features

Relative URLs inside your XProc script (and inside the XSLT/XQuery scripts used by the pipeline) work as expected, that is, they are resolved against the location of the document they’re in. This is an important feature because it allows you to develop your XProc pipelines outside of eXist and easily integrate them when they’re ready. 

Developing XProc pipelines outside of eXist (using an IDE like for instance oXygen) is usually much easier than doing the same thing directly in/on the database.

The XProc `<p:store>` step works and can write documents to the database. The URL must be an absolute URL pointing to a location in the database. **Note:** This URL must begin with the prefix `xmldb://` (instead of the usual `xmldb:exist://`).

## Known limitations

* Probably the most important limitation is that the XQuery scripts called/used by the XProc pipelines do *not* use eXist’s XQuery engine but instead rely on Saxon inside Calabash. As a consequence, you cannot easily access the database or use eXist’s extension functions. XPath instructions will not use any indexes and will not be optimized. 
* The `<pxp:zip>` instruction does not work properly (but the `<pxp:unzip>` does).
