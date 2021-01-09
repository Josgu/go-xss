# go-xss

xss is a module used to filter input from users to prevent XSS attacks. ([What is XSS attack?](http://baike.baidu.com/view/2161269.htm))


[中文说明](./README.md)

[go-xss godoc](https://godoc.org/github.com/feiin/go-xss#example-package)

## Features
Specifies HTML tags and their attributes allowed with whitelist
Handle any tags or attributes using custom function.

## Install

```
go get -u github.com/feiin/go-xss

```


## Usages

```golang

import (
    "github.com/feiin/go-xss"
)


source := "<a href=\"javascript:alert(/xss/)\" title=\"hi\">link</a>"

safeHtml := xss.FilterXSS(source,xss.XssOption{})

```

```golang

import (
    "github.com/feiin/go-xss"
)

source := "<a href=\"javascript:alert(/xss/)\" title=\"hi\">link</a>"
x := xss.NewXSS(xss.XssOption{})
safeHtml := x.Process(source)

```

## Custom filter rules

When using the xss, the options parameter could be used to specify custom rules:


```golang
source := "<a href=\"javascript:alert(/xss/)\" title=\"hi\">link</a>"

options := xss.XssOption{}
safeHtml := xss.FilterXSS(source,options)

```

To avoid passing options every time, you can also do it in a faster way by creating a NewXSS instance:




```golang
source := "<a href=\"javascript:alert(/xss/)\" title=\"hi\">link</a>"
options := xss.XssOption{}

x := xss.NewXSS(options)
safeHtml := x.Process(source)
```

## Whitelist

By specifying a whiteList, e.g. `map[string][]string`. Tags and attributes not in the whitelist would be filter out. For example:

```golang
// only tag a and its attributes href, title, target are allowed
options.WhiteList = map[string][]string {
		"a":{"href","title","target"},
}
// With the configuration specified above, the following HTML:
// <a href="#" onclick="hello()"><i>Hello</i></a>
// would become:
// <a href="#">&lt;i&gt;Hello&lt;/i&gt;</a>
```
For the default whitelist, please refer `xss.GetDefaultWhiteList()`


## Customize the handler function for matched tags

By specifying the handler function with OnTag:

```golang
func onTag(tag, html string, options TagOption) *string {
    // tag is the name of current tag, e.g. 'a' for tag <a>
  // html is the HTML of this tag, e.g. '<a>' for tag <a>
  // options is some addition informations:
  //   isWhite    boolean, whether the tag is in whitelist
  //   isClosing  boolean, whether the tag is a closing tag, e.g. true for </a>
  //   position        integer, the position of the tag in output result
  //   sourcePosition  integer, the position of the tag in input HTML source
  // If a string is returned, the current tag would be replaced with the string
  // If return nil, the default measure would be taken:
  //   If in whitelist: filter attributes using onTagAttr, as described below
  //   If not in whitelist: handle by onIgnoreTag, as described below
}
```


## Customize the handler function for attributes of matched tags


By specifying the handler function with OnTagAttr:

```golang
func OnTagAttr(tag, name, value string,isWhiteAttr bool) *string {
	 // tag is the name of current tag, e.g. 'a' for tag <a>
  // name is the name of current attribute, e.g. 'href' for href="#"
  // isWhiteAttr whether the attribute is in whitelist
  // If a string is returned, the attribute would be replaced with the string
  // If return nil, the default measure would be taken:
  //   If in whitelist: filter the value using safeAttrValue as described below
  //   If not in whitelist: handle by onIgnoreTagAttr, as described below
}

```

## Customize the handler function for tags not in the whitelist

By specifying the handler function with OnIgnoreTag:

```golang
func OnIgnoreTag(tag, html string, options TagOption) *string {
// Parameters are the same with onTag
  // If a string is returned, the tag would be replaced with the string
  // If return nil, the default measure would be taken (specifies using
  // escape, as described below)
}
```

## Customize the handler function for attributes not in the whitelist

By specifying the handler function with onIgnoreTagAttr:

```golang
func OnIgnoreTagAttr(tag,name, value string,isWhiteAttr bool) *string {
  // Parameters are the same with onTagAttr
  // If a string is returned, the value would be replaced with this string
  // If return nil, then keep default (remove the attribute)
}
```


## Customize escaping function for HTML


By specifying the handler function with escapeHtml. Following is the default function (Modification is not recommended):

```golang
func EscapeHTML(html string) string {
	return regGT.ReplaceAllString(regLT.ReplaceAllString(html,"&lt;"),"&gt;")
}

```

## Customize escaping function for value of attributes


By specifying the handler function with safeAttrValue:

```golang
func SafeAttrValue(tag, name, value string) string {
  // Parameters are the same with onTagAttr (without options)
  // Return the value as a string

}
```



### Quick Start

#### Filter out tags not in the whitelist

By using `StripIgnoreTag` parameter:

* `true` filter out tags not in the whitelist
* `false`: by default: escape the tag using configured `escape` function

Example:

If `StripIgnoreTag = true` is set, the following code:

```html
code:<script>alert(/xss/);</script>
```

would output filtered:

```html
code:alert(/xss/);
```

#### Filter out tags and tag bodies not in the whitelist

By using `StripIgnoreTagBody` parameter:

* `nil` by default: do nothing
* `[]string{}`: (empty array) filter out all tags not in the whitelist
* `[]string{"tag1", "tag2"}`: filter out only specified tags not in the whitelist

Example:

If `StripIgnoreTagBody = []string{"script"}` is set, the following code:

```html
code:<script>alert(/xss/);</script>
```

would output filtered:

```html
code:
```

#### Filter out HTML comments

By using `AllowCommentTag` parameter:

* `true`: do nothing
* `false` by default: filter out HTML comments

Example:

If `AllowCommentTag = false` is set, the following code:

```html
code:<!-- something --> END
```

would output filtered:

```html
code: END
```

## Examples

### Allow attributes of whitelist tags start with `data-`

```golang

	source := "<div a=\"1\" b=\"2\" data-a=\"3\" data-b=\"4\">hello</div>";

	html := xss.FilterXSS(source,xss.XssOption{
			OnIgnoreTagAttr: func(tag,name, value string,isWhiteAttr bool) *string {
				if len(name)>=5 && name[0:5] == "data-" {
					ret := name + "=\"" + xss.EscapeAttrValue(value)+"\""
					return &ret
				}
				return nil
			},
	})
	fmt.Printf("%s\nconvert to:\n%s", source, html);
```

Result:

```html
<div a="1" b="2" data-a="3" data-b="4">hello</div>
convert to:
<div data-a="3" data-b="4">hello</div>
```

### Allow tags start with `x-`

```
	source := "<x><x-1>he<x-2 checked></x-2>wwww</x-1><a>";

	html := xss.FilterXSS(source,xss.XssOption{
			OnIgnoreTag: func(tag, html string, options xss.TagOption) *string {
				if len(tag)>=2 && tag[0:2] == "x-" {
					return &html;
				}
				return nil
			},
	})
	fmt.Printf("%s\nconvert to:\n%s", source, html);

```

Result:

```html
<x><x-1>he<x-2 checked></x-2>wwww</x-1><a>
convert to:
&lt;x&gt;<x-1>he<x-2 checked></x-2>wwww</x-1><a>
```

### Parse images in HTML

```golang
	source := "<img src=\"img1\">a<img src=\"img2\">b<img src=\"img3\">c<img src=\"img4\">d"
	var list []string
	html := xss.FilterXSS(source,xss.XssOption{
			OnTagAttr: func(tag, name, value string ,isWhiteAttr bool) *string {
				if tag == "img" && name == "src" {
					list = append(list,value)
				}
				return nil
			},
	})
	fmt.Printf("image list:\n%s", strings.Join(list, ","));
```

Result:

```html
image list:
img1, img2, img3, img4
```

### Filter out HTML tags (keeps only plain text)

```golang
	source := "<strong>hello</strong><script>alert(/xss/);</script>end"
	html := xss.FilterXSS(source,xss.XssOption{
			WhiteList:map[string][]string{},  // 白名单为空，表示过滤所有标签
			StripIgnoreTag:true, // 过滤所有非白名单标签的HTML
			StripIgnoreTagBody:[]string{"script"}, //script标签较特殊，需要过滤标签中间的内容
	})
	fmt.Printf("text: %s", html);
```

Result:

```html
text: helloend
```


## License

MIT