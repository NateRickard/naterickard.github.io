---
layout: post
title: Taming the Linker with XML Config Files
tags: 	xamarin ios tips storyboard xcode C#
excerpt: While there are other ways to control the behavior of the linker, some of these can feel hacky and/or too 'broad' (e.g. telling the linker to skip an entire assembly).  Fortunately, there is another option - a custom linker configuration (XML) file that can be used to control the linker to a very fine degree.
---

I recently worked with a customer who wanted to address the problem of the Xamarin linker removing types, events, and methods.  The linker, which is used to remove unnecessary code to improve app size and overall build times, can sometimes cause issues in user code and also some very popular libraries, [such as MVVM Light](http://blog.galasoft.ch/posts/2015/03/solving-the-event-not-found-issue-in-xamarin-mvvmlight-binding-and-commanding/).

While there are [other ways to control the behavior of the linker](https://developer.xamarin.com/guides/ios/advanced_topics/linker/), some of these can feel hacky and/or too 'broad' (e.g. telling the linker to skip an entire assembly).  Fortunately, there is another option - a custom linker configuration (XML) file that can be used to control the linker to a very fine degree.

The Xamarin [documentation](https://developer.xamarin.com/guides/cross-platform/advanced/custom_linking/) does a great job of defining how to create the XML file and configure a Xamarin project to use it during the build process.  However, actually finding the method/field/event/type names/signatures to include can be a bit of a mystery (and as of this writing, is very lightly documented).

## An Example

Using the MVVM Light library example, we might write this code (Xamarin.Android):

```c#
var button = FindViewById<Button> (Resource.Id.myButton);
button.SetCommand ("Click", ViewModel.MyCommand);
```


If the `Click` event is not preserved by direct use (or any other means), when this code executes in a _Release_ build, you would see an exception similar to:

```
System.ArgumentException
Event not found: Click
Parameter name: eventName
```

So, how might we preserve the needed event using a custom linker config file?

Here's an example file that does just that:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<linker>
	<assembly fullname="Mono.Android">
		<type fullname="Android.Views.View">
			<method name="add_Click" />
			<method name="remove_Click" />
		</type>
	</assembly>
</linker>
```

If I paste that XML into an XML file within my Xamarin.Android project and set its build action to `LinkDescription`, we would see that the linker will now leave the `Click` event in place and the exception will not occur.

But again, how do we determine those method names and/or signatures?

## Determining What to Include

The basic steps I take to figure out the proper names are:

- Find out which class actually defines the event/method/field you want to preserve.
	- One way I've found to locate these easily is to right click the event/method and select "Go to Definition" within the IDE.  Since Xamarin Studio and/or Visual Studio for Mac supports .NET decompilation out of the box, it will take you to the class definition within the assembly where the member is defined.
		- If not using Xamarin Studio/VS for Mac, you could use a tool like dotPeek or just review the API documentation for the member.
	- Once the class & member are found, grab the full namespace + class of that Type.  
	
		Examples:
	
		- In the case of iOS, the `TouchUpInside`, etc., events are actually defined on the `UIKit.UIControl` class rather than `UIButton`, so I would use `<type fullname="UIKit.UIControl">` as the type definition in that case.
		- In the above example, I've located the `Click` event on the `Android.Views.View` namespace+class within the `Mono.Android` assembly.

- At this point you should have all you need for your assembly + type definition:

	```xml
	<assembly fullname="Mono.Android">
	        <type fullname="Android.Views.View">
	...
	```
	
- Next, you'll need to find out how the method or event name shows up **_in the IL_** once compiled.
	- This is because the linker functions at the IL level and needs to match elements with those names, which can and will differ than how they appear in the C# code.
- Finding the correct name(s) depends on what kind of member it is:
  -  Constructors: refer to the naming/syntax [listed in the docs here](https://developer.xamarin.com/guides/cross-platform/advanced/custom_linking/).
  -  Properties: these get rendered as methods once compiled to IL, so they will always show up as a combination of two methods: `get_<PropertyName>` and `set_<PropertyName>`.
  -  Events: this took some digging, but it appears that like properties above, they will show up in the IL as `add_<PropertyName>` and `remove_<PropertyName>` methods in the IL.
  -  Fields: these come through to the IL _as is_, I believe.

As shown in the docs, you can also use more advanced pattern and wildcard matching to find and exclude members from linking as well.

## Wrapping it up

As you can see, it's not too difficult to control the linker via an XML config file.  It's not as fast as some other, more blunt, methods we have available; however, for heavily used libraries or repeatable situations, it's likely one of the best fits due to its granularity and potential for reuse.