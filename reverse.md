# GenMyModel Reverse Engineering Documentation

This document gathers information about the reverse engineering service, the way to control the model production and the Frequently Asked Questions about it.

## Create a model from your source code

GenMyModel reverse engineering service works in a static mode. **The reverse engine gets your sources from github or bitbucket, parses them and produces an UML
model reflecting your code. It never compiles your source code and load it in a JVM.**

To perform a reverse analysis on one of your git repository, go to "New Project from Java" (from the "Project" page, just near the "New project button"). A new
page will open with, basically, the same information about the "New project" page with new information about your repository. Simply put your git repository
URL, give the branch you want to reverse (_master_ by default) and your credentials if the repository is private. Finally, choose your project visibility
and click on `Start reverse`.

## Select the parts of your source code that need analysis

By default, the reverse service scans your whole repository and performs analysis on the `*.java` files it founds. Obviously, sources could (should?) 
include unit tests for example or old code that does not need to be part of the analysis. Also, the part of the sources that needs to be reversed is located 
to a specific directory in your repository. To help you select the files you want, the reverse engineering service provides two 
kinds of entries, **Folders to include** and **Files and Folders to exclude**. The first kind is a string, indicating from the repository root the path to the 
directory that needs to be included in the analysis. For examples, if you want to **include** all the java files under two specific locations: 
```
src/main/java/packageA
src/main/java/packageB
```
With these entries, the analysis will be launched only on the java classes found in `src/main/java/packageA`, `src/main/java/packageB` and their 
sub-directories. Of course, you can add more than two entries.

If you want to **exclude** directories/files, you have to deal with regular expressions. For example, if you want to exclude 
all the unit tests from the analysis and all java classes ending with `Test`:
```
.*/src/test/java/.*
.*Test.java
``` 
Once again, as for directory/file inclusions, you can enter many regexps.

## Controlling reverse engineering from the dashboard

GenMyModel reverse engineering service enables you to control the way your model is built using options (_e.g,_ you can remove methods
from the reversed model or attributes). Keep in mind that these options impact the created model and not your source code! Here is a list 
of the options you have to control the reverse from the dashboard:
* **Remove operations**, _false by default_, removes all the class/interface operations (methods) from the model,
* **Remove attributes**,  _false by default_, removes all the class/interface attributes from the model (except those involved in associations),
* **Discover associations**, _true by default_, automatically discovers associations between your classes/interfaces.
* **Arrays as cardinality**, _false by default_, produces a [0..\*] cardinality for tables (_e.g,_ from `int[][] myTab`, an attribute `myTab : Integer[] [0..*]` 
will be produced),
* **Reduce arrays**, _false by default_, reduces tabs of tabs of tabs... and produces a [0..\*] cardinality instead (_e.g,_ from `int[][] myTab`, an attribute
`myTab : Integer [0..*]` will be produced),
* **Follow collection bindings**, _false by default_, reduces nested collection bindings and produces a [0..\*] (_e.g,_ from `Set<Set<String>> inner` an attribute
`inner : String [0..*]` will be produced with `unique` attribute set to `true`),
* **Add binding expression**, _see below for default value_, enables you to add binding expressions that are used to compute cardinality and reduce them.

The binding expressions follow this format: `matchingExpr, #binding, unique?`, for example, here is the binding expression used to follow lists and lists-like:
```
^[^<]*List.*<.*>, 1, false
```
The `matchingExpr` is a regular expression that should match the type you want to reduce. `#binging` is the entry that needs to be followed and considered as type 
(_e.g,_, for a type `MyType<V1, V2>`, if you want `V2` as type for your element, you have to use `2` as `#binding`). Finally, `unique?` indicates if the produced 
attribute/parameter should  be considered as unique or not.

Here is an other example: suppose you have in your source code a `JsArray<String>`, if you want to reduce this array, you can use the following binding expression: 
`^JsArray<.*>, 1, false`. With this binding, the following attribute `JsArray<String> jsarray` will be reversed as `jsarray : String [0..*]`.


## Combining options (examples)

All the above options can be combined and each option can impact the others. For example, with the previous binding expression:
`^JsArray<.*>, 1, false`, **without** the _follow collection bindings_ option and **with** the _discover associations_ option. Here is 
what happens if this attribute is found in the source code `JsArray<String> tab`. In the reversed model, an attribute will be created as this:
`tab : String [0..*]` (as we previously seen). However, if this attribute is found in the source code `JsArray<MyClass> tab2`, an association 
with one of its cardinality set to [0..\*] will be created in the reversed model.

Here is another example: by combining the _remove attributes_ option **with** the _discover associations_ option, the attributes which will be part of associations are 
let in the model whereas the attribute not involved in associations are removed. Thus, considering this two attributes:
```
String name;
MyClass cl;
```
in the final model, the `name` attribute will be absent from the model and the `cl` attribute will exist as part of an association from the class containing `cl` to 
`MyClass`. By combining the options, you can "shape" the produced model so it fits your requirements.


## Controlling reverse engineering from source code

You can also gain a fine grain control over the model that will be created by the reverse operation. This fine grain mechanism is provided by annotations you add in 
the javadoc. Using options, you can impact general behaviors of the reverse engine, but using annotations, you can take control of a small part of your code and impact 
how it would be reversed.

Currently, there is two annotations you can add in you javadoc (case sensitive):
* **@noreverse**, disable the reverse process for this class/attribute/method,
* **@association**, marks the attribute as an explicit association (even if the _discover associations_ is not use, the attribute will be created as part of an association).

Examples:
```java
public class A {
	/**
	 * @noreverse
	 */
	public final static String TECH_KEY = "Mykey";
	
	/**
	 * @association
	 */
	private MyClass link;

}
```
The produced model will contain a class A and an association from A to MyClass. The reverse had been disabled on the `TECH_KEY` attribute as it represents a pure technical element
and should not be part of the design. The `@association` annotation can take two "arguments" (not required):
* `name`, that sets the association name,
* `opposite`, that sets the opposite attribute in the association.

Lets illustrate how these parameters work:
```java
public class A {
	/**
	 * @noreverse
	 */
	public final static String TECH_KEY = "Mykey";
	
	/**
	 * @association opposite=toA name=myLink
	 */
	private MyClass link;

}

public class MyClass {
	/**
	 * @association opposite=link
	 */
	private A toA;
}
```
The produced model will contain a class A and a bidirectionnal association from A to MyClass named `myLink`. Note that only one `@association` annotation is required 
and the other one could be removed. The following code will give the same result as the previous one.
```java
public class A {
	/**
	 * @noreverse
	 */
	public final static String TECH_KEY = "Mykey";
	
	/**
	 * @association opposite=toA name=myLink
	 */
	private MyClass link;

}

public class MyClass {
	private A toA;
}
```


## Frequently Asked Questions

### Should my source code be compilable/complete?
It doesn't have to. The reverse engine parses your sources but it does not compile/execute your code. Obviously, your code has to conform to the Java grammar, but if a 
part of your class/method... is erroneous, this part is simply removed from the analysis. For example, in the following code:
```
public class A {
	
	priv void faulty() {
	
	}
}
```
the class `A` will be reversed but the method `faulty` will be removed from the analysis. 

### What is the `genmymodel-reverse` package?
Java source code often uses many library classes, types... During the reverse process, these types are not identified as part of your code, but as part of an external library. 
In order to add these types/classes to your model, the engine store them in this package as Datatype/Class/Interface. If it can't guess the imported type kind, it marks it as 
Datatype.

### Why there is `Unknown.xxxx` datatypes/interfaces/classes?
You have probably notice that sometimes, there is some Datatypes/Classes/Interfaces named `Unknown.xxxx` in the `genmymodel-reverse` package. As the engine does not compile/execute your
code, it does not load it in a JVM. In order to deal with imports from a class to other types, the engine try to resolve them the best it can, but with wildcard imports the 
job is not easy. For example:
```java
import org.apache.http.client.*;
import org.apache.commons.io.*;

public class A {
	private HttpClient client;
	private static FileUtils utils = FileUtils;
}
```
In this case, the engine will try to guess from where each type comes from, by questioning a JVM (with a limited number of jar loaded into, keep in mind, that the engine cannot 
load all the (library, version) existing in the world). If it fails, it will creates a `Unknown.HttpClient` Datatype and a `Unknown.FileUtils` Datatype in the `genmymodel-reverse` 
package. To help the engine to correctly resolve the imports, you can change your code to:
```java
import org.apache.http.client.HttpClient;
import org.apache.commons.io.FileUtils;

public class A {
	private HttpClient client;
	private static FileUtils utils = FileUtils;
}
```

However, in simple case where only one wildcard import is found, the entries are correctly resolved.
```java
import org.apache.http.client.*;

public class A {
	private HttpClient client;
}
```
In this example, the created Datatype in the `genmymodel-reverse` package will be `org.apache.http.client.HttpClient`.

Also, regarding your code, the reverse engine could _upgrade_ some of your `Unknown.xxx` datatypes. Indeed, if it founds that one of this datatype is used as part of an `extends`
in a Class, it will upgrade the datatype to a class. If the datatype is used as part of an `extends` in an interface, the engine will upgrade this datatype to an interface. Finally,
if the datatype is used as part of an `implements`, it will upgrade the datatype to interface.

### How can I deal with elements contained in the `genmymodel-reverse` package with my own generator? Will they be generated as code?
During code generation, these elements are split from your model and temporarily stored in another model/resource. This way, they are no longer
part of your model during generation (only during generation process). You can still access them and handle them, but as they are in a
different model, unless you write in your generator that your want to generate code for them, they will not be translated to code.

### Why the reverse process is taking so long?
The reverse engineering process performs 4 main phases:

1. your repository is cloned,
1. analysis is performed on your code and a first model is produced,
1. the produced model is refined,
1. the cloned repository is deleted.

During benchmarks, we observed that in almost 95% of the time, the repository cloning operation takes the most time. The engine has to get your entire code and git history from your
repository and, obviously, if your history is long and big, well... you do the math ;) (of course, we are working on other solution to accelerate this first phase).
